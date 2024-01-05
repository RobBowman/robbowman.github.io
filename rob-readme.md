# Running Jekyll in a Dev Container

This guide outlines the steps to run a Jekyll site inside a Docker container using Visual Studio Code (VSCode).

## Steps to Run Jekyll in the Container

### 1. Open the Project in the Container

- Open your Jekyll project in VSCode.
- Use the command "Dev-Containers: Open Folder in Container..." which is available in the Command Palette (`F1` or `Ctrl+Shift+P`).

### 2. Access the Container's Terminal

- Once the project is open in the container, open a new terminal in VSCode. This terminal will be inside the container.

### 3. Navigate to the Project Directory

- Ensure that you are in your project's root directory, where your Jekyll site's files are located.

### 4. Run Jekyll

- Run the following command to start the Jekyll server inside the container:
  ```bash
  bundle exec jekyll serve

### Nativate to the Site
http://localhost:4000/