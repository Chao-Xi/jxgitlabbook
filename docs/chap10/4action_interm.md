


### Avoiding Duplication by Using Reusable Workflows

1. **Avoiding Duplication with GitHub Actions Workflows**:
   - To prevent duplication in GitHub Actions workflows, utilize reusable workflows, composite actions, and starter workflows.
  
2. **Reusable Workflows**:
   - Reusable workflows are entire workflows that can be called by other workflows, stored in the same repository, or a central shared repository.
   - Reusable workflows can have one or multiple jobs and are called as separate jobs, not just as steps within a job.
   - They show each step in the workflow in the log file output, unlike composite actions.
  
3. **Composite Actions**:
   - Composite actions combine multiple steps into a single action, aiding in refactoring long YAML scripts and avoiding duplication.
   - They are a series of steps that can be called from within a job in another workflow.

4. **Starter Workflows**:
   - Starter workflows are templates that provide a starting point for creating workflows, making it quicker and easier for teams to set up workflows for their projects.
   - They allow modifications for specific project requirements.

5. **Calling Reusable Workflows**:
   - To call a reusable workflow stored in another repository, specify the owner name, repo name, and path to the workflow starting with .github/workflows.
   - For reusable workflows in the same repo, use the relative path and call it using the file name.
   - Reusable workflows run as part of the caller workflow, allowing actions like checkout to work on the caller's code.

6. **Creating Reusable Workflows**:
   - When creating a reusable workflow, define the trigger in the 'on' section as 'workflow_call'.
   - Pass variables, including secrets, from the caller workflow's repository to the reusable workflow for effective integration.

7. **Focus on Reusable Workflows**:
   - The article primarily focuses on reusable workflows, demonstrating their utility and how to implement them effectively for workflow optimization.

