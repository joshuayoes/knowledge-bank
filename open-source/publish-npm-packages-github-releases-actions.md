# How to Publish npm Packages using GitHub Releases and Actions

This guide outlines how to set up a project to automatically publish npm packages whenever you create a GitHub Release. 

## When should I use this?

There are much more featured solutions, but this a low weight option if want to minimize your dependencies in your project or don't want to adopt the opinions of other tools. This is used on [ios-simulator-mcp](https://github.com/joshuayoes/ios-simulator-mcp/) since it is a simple server in an ecosystem that doesn't not have an existing convention for releasing (at the time of writing this).

## 1. Initialize Git Repository

For this guide, we'll use `react-native-awesome-library` as our example library name. Start by creating a new directory for your library and initializing a Git repository.

```bash
mkdir react-native-awesome-library
cd react-native-awesome-library
git init
```

## 2. Initialize npm Package

Next, initialize your project as an npm package.

```bash
npm init -y
# Answer the prompts or use `npm init -y` for defaults.
# Make sure your package name is `react-native-awesome-library` (or your desired name).
```

This creates your `package.json` file.

## 3. `package.json` Setup

Edit your `package.json` to include essential fields and a build step.

*   **`name`**: Should match your intended npm package name (`react-native-awesome-library`).
*   **`version`**: Start typically at `0.1.0`.
*   **`main`**: The entry point file for your library (e.g., `lib/index.js`).
*   **`files`**: An array of files and directories to include in the published package (e.g., `["lib/", "README.md"]`).
*   **`scripts`**: Add `build` and `prepare` scripts. The `prepare` script runs automatically before npm publishes.

```jsonc
// package.json
{
  "name": "react-native-awesome-library",
  "version": "0.1.0",
  "description": "An awesome React Native library.",
  "main": "lib/index.js", // Or your main entry point
  "files": [
    "lib", // Directory containing built code
    "README.md",
    "LICENSE"
  ],
  "scripts": {
    "build": "tsc", // Or your build command (e.g., babel, webpack)
    "prepare": "npm run build" // Runs build before publishing
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/YOUR_USERNAME/react-native-awesome-library.git" // Replace with your repo URL
  },
  // ... other fields like keywords, author, license, dependencies, devDependencies
}
```

Make sure your build process outputs the necessary files to the directories listed in the `files` array (e.g., the `lib` directory).

## 4. GitHub Workflow Setup

Create a GitHub Actions workflow file to handle the publishing.

1.  **Create Workflow File:** Create `.github/workflows/publish.yml`:

    ```yaml
    # .github/workflows/publish.yml
    name: Publish Package to npm

    on:
      release:
        types: [published] # Triggers when a release is published on GitHub

    jobs:
      build-and-publish:
        runs-on: ubuntu-latest
        permissions:
          contents: read
          id-token: write # Required for provenance
        steps:
          - uses: actions/checkout@v4

          # Setup Node.js
          - uses: actions/setup-node@v4
            with:
              node-version: "20.x" # Or your preferred Node.js version
              registry-url: "https://registry.npmjs.org" # Sets up .npmrc

          # Install dependencies
          - run: npm ci # Use ci for clean installs in CI

          # Build step is implicitly run by `npm publish` via the `prepare` script

          # Publish to npm
          - run: npm publish --provenance --access public
            env:
              NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }} # Use the secret token
    ```

2.  **Add npm Token to GitHub Secrets:**
    *   Generate an npm access token with "Automation" permissions on the [npm website](https://www.npmjs.com/settings/YOUR_NPM_USERNAME/tokens).
    *   Go to your GitHub repository -> Settings -> Secrets and variables -> Actions.
    *   Click "New repository secret".
    *   Name the secret `NPM_TOKEN`.
    *   Paste your npm access token as the value.

## 5. Initial Release

Now you're ready for the first release.

1.  **Commit and Push:** Make sure all your code, `package.json`, and the workflow file are committed and pushed to GitHub.
    ```bash
    git add .
    git commit -m "feat: Initial project setup for react-native-awesome-library"
    git push origin main # Or your default branch
    ```
2.  **Create GitHub Release:**
    *   **UI Method:** Go to your repository on GitHub -> Releases -> "Draft a new release".
        *   Click "Choose a tag", type `v0.1.0` (matching your `package.json` version), and click "Create new tag: v0.1.0 on publish".
        *   Add a release title (e.g., `v0.1.0 - Initial Release`).
        *   Write release notes.
        *   Click "Publish release".
    *   **CLI Method (`gh`):**
        ```bash
        # Ensure package.json version is 0.1.0
        git tag v0.1.0
        git push origin v0.1.0
        gh release create v0.1.0 --title "v0.1.0 - Initial Release" --notes "First release of react-native-awesome-library"
        ```
3.  **Check Workflow:** Go to the "Actions" tab in your GitHub repository. You should see the "Publish Package to npm" workflow running or completed.
4.  **Verify on npm:** Check [npmjs.com](https://www.npmjs.com/) for your package (`react-native-awesome-library`) to confirm it was published successfully.

## 6. Subsequent Releases

For future releases:

1.  **Update Code:** Make your changes and improvements.
2.  **Bump Version:** Update the `version` field in `package.json` according to [Semantic Versioning](https://semver.org/) (e.g., `0.1.1`, `0.2.0`, `1.0.0`).
3.  **Commit and Push:** Commit the code changes and the `package.json` update.
    ```bash
    git add .
    git commit -m "feat: Add new feature X" # Or fix:, chore:, etc.
    git push origin main
    ```
4.  **Create GitHub Release:** Follow the same process as the initial release (UI or CLI), but use the new version number for the tag and title (e.g., `v0.1.1`).
    ```bash
    # Example using gh cli for v0.1.1
    # Ensure package.json version is 0.1.1 first
    git tag v0.1.1
    git push origin v0.1.1
    gh release create v0.1.1 --title "v0.1.1 - Added Feature X" --notes "Implemented feature X to enhance the library."
    ```
5.  **Check Workflow & Verify:** The GitHub Action will trigger again, publishing the new version. Verify on the Actions tab and npmjs.com.

That's it! You now have an automated npm publishing pipeline tied to your GitHub Releases.