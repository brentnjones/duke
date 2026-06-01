# Onboarding New Application Documentation

Use this guide when you want to add documentation for a new application to the document library.

## What to Add

For each application, add documentation that covers:

- Purpose of the application
- Ownership and support contact
- Deployment or hosting details
- Operational runbooks
- Troubleshooting steps
- Links to source repositories, dashboards, or tickets

## Recommended Structure

Create a folder for the application under the `docs/` directory and keep related files together.

Example:

```text
docs/
  my-application/
    overview.md
    runbook.md
    troubleshooting.md
```

## Onboarding Steps

1. Create a folder for the application under `docs/`.
2. Add an overview page that explains what the application does.
3. Add supporting operational documentation such as runbooks and troubleshooting guides.
4. Update `mkdocs.yml` so the new pages appear in TechDocs navigation.
5. Add a link from `docs/index.md` if you want the application visible from the main landing page.
6. Commit and push the changes to the repository.

## Review Checklist

- The docs are written in plain Markdown.
- File names are clear and consistent.
- Links work correctly.
- The application owner is identified.
- TechDocs navigation includes the new pages.
