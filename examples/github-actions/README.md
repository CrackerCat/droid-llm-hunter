# Droid LLM Hunter - GitHub Action

This action allows you to integrate **Droid LLM Hunter** directly into your GitHub Actions workflow to automatically scan Android applications (APKs) for vulnerabilities using AI.

## Usage

### Prerequisites

1.  **Build your APK**: Your workflow must build the APK first (e.g., using `gradlew`).
2.  **API Key**: You need an API key for your chosen LLM provider (Gemini, OpenAI, etc.) stored in GitHub Secrets.

> [!IMPORTANT]
> **Provider Recommendation**: For CI/CD (GitHub Actions), we strictly recommend using **Cloud APIs** (Gemini, Groq, OpenAI).
> Using **Ollama** is not recommended on standard GitHub Runners because they lack GPUs and require complex setup to run the Ollama service, leading to timeouts or failures.

### Example Workflow

Create a file `.github/workflows/security.yml` in your repository:

```yaml
name: Security Scan

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]
  schedule:
    - cron: "0 0 * * *" # Run nightly

jobs:
  scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: "17"
          distribution: "temurin"

      - name: Build with Gradle
        run: ./gradlew assembleDebug

      - name: Droid LLM Hunter Scan
        uses: roomkangali/droid-llm-hunter@v1.1.6 # Replace with tag/version if available
        with:
          apk-path: app/build/outputs/apk/debug/app-debug.apk
          provider: "gemini"
          model: "gemini-2.0-flash" # Recommended: faster and more efficient
          api-key: ${{ secrets.GEMINI_KEY }}
          # Optional: Customize vulnerability rules
          # rules: sql_injection:true, webview_xss:false, hardcoded_secrets:true

      - name: Upload Report
        uses: actions/upload-artifact@v4
        with:
          name: security-report
          path: droid-llm-report.json
```

## Inputs

| Input         | Description                                                                                              | Required | Default          |
| ------------- | -------------------------------------------------------------------------------------------------------- | -------- | ---------------- |
| `apk-path`    | Path to the APK file relative to the repository root.                                                    | **Yes**  | N/A              |
| `provider`    | LLM Provider: `gemini`, `openai`, `groq`, `anthropic`, `openrouter`, `ollama`.                          | No       | `gemini`         |
| `api-key`     | Your API Key. Should be passed via GitHub Secrets (e.g., `${{ secrets.GEMINI_KEY }}`).                  | **Yes*** | N/A              |
| `model`       | Specific model name. **Recommended**: `gemini-2.0-flash`, `gpt-4-turbo`, `claude-sonnet-4-20250514`.    | No       | Provider default |
| `rules`       | Customize vulnerability checks. Format: `rule_name:true/false` (comma-separated). See **Rules** section. | No       | All enabled      |
| `config-path` | Path to custom `settings.yaml` file for advanced configuration.                                          | No       | Built-in config  |

> **\*** `api-key` is required for cloud providers (`gemini`, `openai`, `groq`, `anthropic`, `openrouter`), but optional for `ollama` (local).

## Vulnerability Rules

You can selectively enable/disable specific vulnerability checks using the `rules` parameter. This allows you to focus on specific security concerns or reduce scan time.

### Available Rules (24 Total)

| Rule Name                            | Description                                      | Default |
| ------------------------------------ | ------------------------------------------------ | ------- |
| `sql_injection`                      | Database manipulation via rawQuery/execSQL       | ✅      |
| `webview_xss`                        | JavaScriptInterface & WebView XSS                | ✅      |
| `hardcoded_secrets`                  | API Keys & Secrets hardcoded in code             | ✅      |
| `hardcoded_secrets_xml`              | Secrets in strings.xml or resources              | ✅      |
| `insecure_storage`                   | Plain-text credentials in files                  | ✅      |
| `insecure_webview`                   | Dangerous WebSettings (FileAccess, JS enabled)   | ✅      |
| `exported_components`                | Activities/Services exposed publicly             | ✅      |
| `insecure_deserialization`           | RCE via ObjectInputStream                        | ✅      |
| `intent_spoofing`                    | Unauthorized Broadcast Receivers                 | ✅      |
| `path_traversal`                     | Local File Inclusion via ContentProvider         | ✅      |
| `webview_file_access`                | Cookie theft via local file access               | ✅      |
| `webview_deeplink`                   | Modern XSS vector & Open Redirect                | ✅      |
| `deeplink_hijack`                    | Missing autoVerify in Intent Filters             | ✅      |
| `deeplink_logic_bypass`              | IDOR/Auth bypass via Deep Links                  | ✅      |
| `pending_intent_hijacking`           | Mutable PendingIntent privilege escalation       | ✅      |
| `unsafe_reflection`                  | RCE via Class.forName/Method.invoke              | ✅      |
| `insecure_file_permissions`          | World-readable files/SharedPreferences           | ✅      |
| `insecure_random_number_generation`  | Weak PRNG (java.util.Random)                     | ✅      |
| `zip_slip`                           | Path traversal during Zip extraction             | ✅      |
| `fragment_injection`                 | PreferenceActivity RCE                           | ✅      |
| `biometric_bypass`                   | Insecure onAuthenticationFailed logic            | ✅      |
| `strandhogg`                         | Task affinity & launch mode hijacking            | ✅      |
| `graphql_injection`                  | Dynamic GraphQL query construction               | ✅      |
| `jetpack_compose_security`           | Debug flags in Compose UI                        | ✅      |
| `universal_logic_flaw`               | LLM-based conceptual logic analysis              | ✅      |

### Example: Custom Rules Configuration

```yaml
# Enable only SQL injection and secret detection
rules: sql_injection:true, hardcoded_secrets:true, hardcoded_secrets_xml:true, insecure_storage:false, webview_xss:false

# Disable specific checks
rules: biometric_bypass:false, strandhogg:false, universal_logic_flaw:false

# Focus on WebView vulnerabilities only
rules: webview_xss:true, insecure_webview:true, webview_file_access:true, webview_deeplink:true, sql_injection:false
```

> **💡 Tip**: If you don't specify `rules`, all vulnerability checks are enabled by default. You can also create a custom `config/settings.yaml` file and reference it using `config-path` parameter for advanced configuration.

## Outputs

The action generates a `droid-llm-report.json` file in the root of the workspace containing:
- **Vulnerability findings** with severity ratings
- **Affected code locations** with line numbers
- **Remediation recommendations**
- **Attack surface analysis**

You should upload this report as an artifact for review:

```yaml
- name: Upload Report
  uses: actions/upload-artifact@v4
  if: always() # Upload even if scan finds issues
  with:
    name: security-report
    path: droid-llm-report.json
    retention-days: 14 # Keep report for 2 weeks
```

## Behavior

This action is **Non-Blocking** by default. It will generate a report but will not fail the build if vulnerabilities are found, unless a critical tool error occurs. This is intentional to account for potential False Positives common in AI-powered analysis.

### Best Practices

1. **Review Reports Regularly**: Download and review the generated `droid-llm-report.json` from GitHub Actions artifacts.
2. **Use Secrets Properly**: Always store API keys in GitHub Secrets (Settings → Secrets and variables → Actions).
3. **Customize Rules**: For faster scans, disable rules that don't apply to your app.
4. **Recommended Model**: Use `gemini-2.0-flash` for optimal speed and accuracy in CI/CD pipelines.
5. **Verify APK Before Scanning**: Add a verification step to ensure APK exists before running the scan (see example workflow).
