# How to Integrate Gemini CLI with n8n (Local)

This guide explains how to enable your local n8n instance (running in Docker) to execute the `gemini-cli` installed on your Mac using SSH.

## Prerequisites
1.  **n8n Running:** You should have n8n running via Docker Compose.
2.  **Gemini CLI Installed:** You have `@google/gemini-cli` installed on your Mac.
    - Verified Path: `/Users/administrator/.npm-global/bin/gemini`

## Step 1: Enable Remote Login (SSH) on Mac

For n8n to connect to your Mac, you must enable SSH access.

1.  Open **System Settings**.
2.  Go to **General** -> **Sharing**.
3.  Toggle **Remote Login** to **ON**.
4.  Click the "i" (Info) button next to Remote Login.
5.  Ensure your user (`administrator`) is in the list of "Allow access for".

## Step 2: Import the Workflow

1.  Open n8n in your browser (http://localhost:5678).
2.  Go to the **Workflows** list.
3.  Click **"Add workflow"** -> **"Import from..."** -> **"File"**.
4.  Select the file: `n8n_gemini_cli_workflow.json` (located in your project folder).

### Alternative Method (Copy & Paste)
If the file import fails:
1.  Open `n8n_gemini_cli_workflow.json` in a text editor.
2.  **Copy** the entire content.
3.  Go to the n8n Canvas (in your browser).
4.  Press **Cmd+V** (Mac) or **Ctrl+V** to paste.
5.  The nodes should appear on your canvas.


## Step 3: Configure SSH Credentials

The imported workflow has a placeholder for SSH credentials. You need to configure them.

1.  Double-click the **Execute Gemini CLI** node.
2.  Under **Authentication**, select **"Predefined Credential Type"**.
3.  For **Credential**, select **"Create New"**.
4.  Fill in the details:
    *   **Host:** `host.docker.internal`
    *   **Port:** `22`
    *   **Username:** `administrator`
    *   **Password:** *Your Mac Login Password* (or configure a Private Key if you prefer).
5.  Click **Save**.

## Step 4: Test

1.  Click **"Test step"** on the **Execute Gemini CLI** node.
    *   **Command:**
        ```bash
        export PATH=$PATH:/usr/local/bin && echo "You are a helpful assistant." > GEMINI.md && /Users/administrator/.npm-global/bin/gemini '{{$json.prompt}}' --output-format json
        ```
    *   **CWD:** `/Users/administrator/gemini-n8n-execution` (Make sure this folder exists!)
3.  The output will show the response from Gemini.

## Troubleshooting

*   **"Connection refused":** Ensure "Remote Login" is actually enabled.
*   **"Permission denied":** Check your username/password.
*   **"Command not found":** Ensure the path `/Users/administrator/.npm-global/bin/gemini` is correct.
*   **"env: node: No such file":** This means `node` is not in the SSH PATH. The workflow command includes `export PATH=$PATH:/usr/local/bin`.
*   **Variable Not Substituting (`{{$json.prompt}}` literal):** Ensure the **Command** field in the n8n node is set to **Expression** mode (toggle the "Fixed/Expression" button or look for the orange variable highlighting). If set to Fixed (String), n8n sends the literal text `{{...}}` to the shell.
*   **Directory Permissions/EPERM:** macOS often restricts SSH access to `Documents`. The workflow now uses `/Users/administrator/gemini-n8n-execution` as a safe working directory. Ensure this directory exists (`mkdir ~/gemini-n8n-execution`).

