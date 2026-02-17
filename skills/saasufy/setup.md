# Saasufy Setup

## What this skill does

This skill guides users through the initial setup of their Saasufy project, ensuring they have the necessary API credentials to interact with their Saasufy account.

## When to use this skill

- When the user first clones this repository
- When the user needs to configure or update their Saasufy API credentials
- When API authentication is failing and credentials need to be verified

## Instructions

### Step 1: Verify API credential file exists

Check if the file `.saasufy-api-key` exists in the project root directory.

If it exists:
- Read the file to verify it contains a valid-looking API key (should be a long hexadecimal string)
- Inform the user that credentials are already configured
- Ask if they want to update or verify their credentials

If it does NOT exist:
- Proceed to Step 2 to guide the user through credential creation

### Step 2: Guide user to create API credentials

Provide the user with these step-by-step instructions:

1. **Create a Saasufy account** (if they haven't already):
   - Go to https://saasufy.com
   - Sign up for an account

2. **Generate an Admin HTTP API credential**:
   - Log into your Saasufy dashboard at https://saasufy.com
   - Navigate to the **Authentication** page
   - Scroll to the bottom of the page
   - Click to create a new **Admin HTTP API credential**
   - Ensure it has **read and write access** (this is the default setting)
   - Once created, **copy the secret key** that is displayed

   **IMPORTANT**: The secret key is only shown once. Make sure to copy it before closing the dialog.

3. **Save the credential**:
   - Once the user has copied their secret key, ask them to provide it
   - When they provide the key, save it to `.saasufy-api-key` in the project root
   - Confirm the credential has been saved successfully

### Step 3: Verify the setup

After saving the credential:
- Inform the user that their API credential has been saved to `.saasufy-api-key`
- Let them know that this file should be kept secure and should NOT be committed to version control
- Verify that `.saasufy-api-key` is listed in `.gitignore` (create or update `.gitignore` if needed)

### Step 4: Ask the user to perform the first deployment

- Ask the user to log into their Saasufy control panel click on the `Deploy service` button on their `Dashboard` page.
- Ask to copy the `Service URL` and provide it to you.
- Once you have their service URL, replace the dummy one from the `.saasufy-service-url` file located in the project root.

### Step 5: Test the connection (optional)

Optionally, you can offer to test the API connection:
- Read the API key from `.saasufy-api-key`
- Make a simple API GET request to verify the Admin credential (for schema managament) works
- Example: `curl -H "Authorization: Bearer <api-key>" https://api.saasufy.com/api/...`
- Also test the user's service URL using a GET request to test the ability to manage data within the service.

## Important notes

- The API key file (`.saasufy-api-key`) should contain ONLY the secret key as plain text, nothing else
- No JSON formatting, no quotes, no extra characters - just the raw key
- This makes it simple for non-technical users and prevents formatting errors
- Always ensure `.saasufy-api-key` is in `.gitignore` to prevent accidental commits
- Other skills will rely on this file existing, so proper setup is critical

## Communication style

- Be friendly and encouraging
- Use clear, simple language suitable for non-technical users
- Provide direct links and specific navigation instructions
- Confirm each step before moving to the next
- Celebrate successful setup
