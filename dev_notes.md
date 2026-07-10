# AWS Credential Strategy for FastAPI

This report explains how the FastAPI backend can connect to AWS services without storing long-lived AWS access keys in application configuration. It covers three deployment scenarios:

1. S3 access from EC2 without access keys.
2. Dynamic application secrets from AWS Secrets Manager.
3. Local developer access from Windows, WSL, Ubuntu, or macOS without sharing access keys.

## Background

DevKraft's updated security policy does not allow AWS access keys and secret keys to be stored in application configuration. The preferred approach is to rely on AWS-managed identity sources such as EC2 instance roles, AWS CLI login, or temporary credentials discovered automatically by boto3.

The backend must support these environments:

- DevKraft development instance.
- Indegene QA, UAT, and production environments.
- Local developer machines.

---

## Use Case 1: Configure S3 Without Access Keys

### Problem

The FastAPI application needs to access S3, but the application should not require `S3_ACCESS_KEY` or `S3_SECRET_KEY` in `.env` for AWS-hosted environments.

### How boto3 Resolves Credentials

boto3 uses the [default credential provider chain](https://docs.aws.amazon.com/boto3/latest/guide/credentials.html). It checks supported credential sources in order and stops as soon as it finds usable credentials.

Common credential sources include:

1. Credentials passed directly to `boto3.client()`.
2. Credentials passed to a `boto3.Session`.
3. Environment variables.
4. Assume-role providers.
5. Web identity providers.
6. AWS IAM Identity Center credentials.
7. Shared credentials file: `~/.aws/credentials`.
8. AWS config file: `~/.aws/config`.
9. Container credential provider.
10. EC2 Instance Metadata Service when the instance has an IAM role.

Important behavior:

- If no explicit access key and secret key are configured, boto3 automatically tries the next available provider.
- If boto3 finds credentials but they are invalid or unauthorized, AWS calls fail instead of continuing to later providers.
- EC2 IAM role credentials are temporary and rotated automatically by AWS.
- Applications running in Docker on EC2 can still use the EC2 instance role through the Instance Metadata Service, assuming network access to metadata is available.

### EC2 IAM Role Flow

For an application running on EC2:

```text
FastAPI -> boto3 -> IMDSv2 -> EC2 IAM Role -> AWS STS -> Temporary Credentials -> S3
```

The EC2 Instance Metadata Service is available at (port 80):

```text
http://169.254.169.254
```

### DevOps Request

Ask DevOps to attach the required S3 permissions to the EC2 instance role.

This must be an EC2 role policy, not an IAM user policy. The policy should allow only the required bucket actions, for example read, write, list, or delete depending on the backend's actual requirements.

### Recommended Implementation

Keep optional explicit credentials for legacy or local fallback use, but do not require them. If access key and secret key are both empty, pass no credential arguments to boto3 and let the default credential chain take over.

```python
_s3_client = None
_s3_client_lock = Lock()


def _credential_kwargs(s) -> Dict[str, str]:
    if bool(s.s3_access_key) != bool(s.s3_secret_key):
        raise RuntimeError(
            "Both S3_ACCESS_KEY and S3_SECRET_KEY must be set, or neither. "
            "Leave both empty to use boto3 default credentials."
        )

    if not s.s3_access_key:
        return {}

    kwargs = {
        "aws_access_key_id": s.s3_access_key,
        "aws_secret_access_key": s.s3_secret_key,
    }
    if s.s3_session_token:
        kwargs["aws_session_token"] = s.s3_session_token
    return kwargs


def get_s3_client():
    global _s3_client

    if _s3_client is not None:
        return _s3_client

    with _s3_client_lock:
        if _s3_client is not None:
            return _s3_client

        s = get_settings()
        if not s.s3_bucket_name:
            raise RuntimeError("S3 configuration missing: S3_BUCKET_NAME")

        region = s.s3_region or _find_bucket_region(s.s3_bucket_name)
        _s3_client = boto3.client(
            "s3",
            **_credential_kwargs(s),
            region_name=region,
            endpoint_url=f"https://s3.{region}.amazonaws.com",
            config=Config(
                signature_version="s3v4",
                s3={"addressing_style": "virtual"},
            ),
        )
        return _s3_client
```

Required `.env` value:

```env
S3_BUCKET_NAME=<bucket-name>
```

Recommended `.env` value:

```env
S3_REGION=ap-south-1
```

Region auto-detection may not work reliably in every environment, so setting `S3_REGION` explicitly is safer.

Source https://docs.aws.amazon.com/boto3/latest/guide/credentials.html
<img width="489" height="316" alt="image" src="https://github.com/user-attachments/assets/b9cb82af-b380-4441-a15b-14fae7f8b86b" />


---

## Use Case 2: Configure Dynamic AWS Secrets Manager

### Problem

The DevKraft development instance should load application secrets from AWS Secrets Manager, while Indegene environments can continue using `.env` values if needed.

Secrets may include OpenAI keys and other external service credentials.

### Approach Options

| Approach | Behavior | Notes |
| --- | --- | --- |
| Direct SDK client | Reads the secret through boto3 when the application starts. | Simple and sufficient for startup configuration. |
| Caching client | Keeps an in-memory cached copy per worker. | Useful when secrets are fetched repeatedly at runtime. |

Current recommendation: load secrets once during application bootstrap. This avoids repeated Secrets Manager API calls and keeps runtime behavior predictable.

### Pricing Note

AWS Secrets Manager pricing is typically based on:

- Monthly storage cost per secret.
- API call cost per number of requests.

A single application-level JSON secret can store multiple key-value pairs, so one secret is usually enough for backend configuration.

Example secret value:

```json
{
  "OPENAI_API_KEY": "...",
  "AZURE_OPENAI_ENDPOINT": "...",
  "AZURE_OPENAI_API_VERSION": "..."
}
```

### Recommended Startup Behavior

Load environment values once during process startup:

1. Load `.env` first.
2. If `AWS_SECRETS_NAME` is configured, fetch that secret from AWS Secrets Manager.
3. Copy each secret key-value pair into `os.environ`.
4. Allow `AWS_SECRETS_OVERRIDE_ENV` to decide whether Secrets Manager values replace existing environment values.

Call `bootstrap_environment()` before settings are initialized in both:

1. `label/main.py`
2. `label/alembic/env.py`

Alembic needs the same bootstrap step because migrations run outside the FastAPI startup lifecycle.

### Environment Variables

```env
AWS_SECRETS_NAME=<secret-name>
AWS_SECRETS_REGION=ap-south-1
AWS_SECRETS_OVERRIDE_ENV=true
```

Fallback region order:

1. `AWS_SECRETS_REGION`
2. `AWS_REGION`
3. `AWS_DEFAULT_REGION`
4. `S3_REGION`
5. `ap-south-1`

### Recommended Implementation

```python
ENV_PATH = Path(__file__).resolve().parent / ".env"
_bootstrap_lock = Lock()
_bootstrapped = False


def _env_str(name: str, default: str = "") -> str:
    value = os.environ.get(name, default).strip()
    if len(value) >= 2 and value[0] == value[-1] and value[0] in {"'", '"'}:
        value = value[1:-1].strip()
    return value or default


def _env_int(name: str, default: int) -> int:
    value = _env_str(name)
    return int(value) if value else default


def _env_float(name: str, default: float) -> float:
    value = _env_str(name)
    return float(value) if value else default


def _env_bool(name: str, default: bool = False) -> bool:
    value = _env_str(name)
    if not value:
        return default
    return value.lower() in {"1", "true", "yes", "on"}


def _load_env_file(path: Path) -> None:
    if not path.exists():
        return

    for key, value in dotenv_values(path).items():
        if not key or value is None or os.environ.get(key):
            continue
        os.environ[key] = str(value)


@lru_cache(maxsize=1)
def _get_aws_secret_payload(secret_name: str, region_name: str) -> dict[str, str]:
    client = boto3.client("secretsmanager", region_name=region_name)
    try:
        response = client.get_secret_value(SecretId=secret_name)
    except ClientError as exc:
        raise RuntimeError(f"Failed to read AWS secret {secret_name}: {exc}") from exc

    secret_string = response.get("SecretString")
    if not secret_string:
        raise RuntimeError(f"AWS secret {secret_name} has no SecretString value.")

    try:
        payload = json.loads(secret_string)
    except json.JSONDecodeError as exc:
        raise RuntimeError(f"AWS secret {secret_name} must be a JSON object.") from exc

    if not isinstance(payload, dict):
        raise RuntimeError(f"AWS secret {secret_name} must be a JSON object.")

    return {str(key): str(value) for key, value in payload.items() if value is not None}


def bootstrap_environment() -> None:
    global _bootstrapped

    if _bootstrapped:
        return

    with _bootstrap_lock:
        if _bootstrapped:
            return

        _load_env_file(ENV_PATH)

        secret_name = _env_str("AWS_SECRETS_NAME")
        if secret_name:
            region_name = _env_str(
                "AWS_SECRETS_REGION",
                _env_str(
                    "AWS_REGION",
                    _env_str(
                        "AWS_DEFAULT_REGION",
                        _env_str("S3_REGION", "ap-south-1"),
                    ),
                ),
            )
            override_env = _env_bool("AWS_SECRETS_OVERRIDE_ENV", True)
            secret_values = _get_aws_secret_payload(secret_name, region_name)
            for key, value in secret_values.items():
                if override_env or not os.environ.get(key):
                    os.environ[key] = value

        _bootstrapped = True
```

### DevOps Request

Ask DevOps to allow the runtime role to read the configured secret:

```text
secretsmanager:GetSecretValue
```

The permission should be scoped to the specific secret ARN rather than all secrets.

---

## Use Case 3: Connect from a Developer Laptop Without Access Keys

### Problem

Developers need to run the backend from a local machine and connect to AWS services such as S3 and Secrets Manager, but they should not use shared long-lived access keys.

### Recommended Local Authentication

Use AWS CLI browser-based authentication. After login, AWS CLI stores temporary credentials in the standard AWS files under the user's home directory. boto3 can discover those credentials automatically through the same default credential chain.

Typical credential locations:

```text
~/.aws/credentials
~/.aws/config
```

For Docker-based local development, mount the host AWS configuration directory into the container so boto3 inside the container can read it.

Example Docker mount:

```yaml
volumes:
  - ~/.aws:/root/.aws:ro
```

Adjust the container path if the application runs as a non-root user.

### DevOps Request

Ask DevOps to assign the `SignInLocalDevelopmentAccess` policy to each developer's personal IAM user.

This is an IAM user policy for local development, not an EC2 instance role policy.

### AWS CLI Installation

#### Windows

Download and install AWS CLI v2:

```text
https://awscli.amazonaws.com/AWSCLIV2-User.msi
```

#### Ubuntu

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

#### macOS

Follow the official AWS CLI installation guide:

```text
https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
```

### Login Steps

1. Confirm AWS CLI is installed:

   ```bash
   aws --version
   ```

2. Login with the approved DevKraft identity flow:

   ```bash
   aws login
   ```

3. Run the backend without setting `S3_ACCESS_KEY` or `S3_SECRET_KEY`.

boto3 will automatically use the local AWS CLI credentials.

---

## Final Recommendation

Use one credential strategy across all environments:

- Do not store long-lived AWS access keys in `.env`.
- Use EC2 instance roles for AWS-hosted environments.
- Use AWS Secrets Manager for external service credentials where DevKraft requires centralized secret management.
- Use AWS CLI login for local development.
- Keep `S3_BUCKET_NAME` and `S3_REGION` as normal configuration values because they are not secrets.

This keeps the application compatible with boto3's standard credential chain while aligning with the no-access-key policy.



