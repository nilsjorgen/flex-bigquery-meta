# AGENTS.md

This workspace is a meta-repository. It contains independent Git repositories
that [meta](https://github.com/mateodelnorte/meta) manages.

## Project structure

The following repositories are NOT searate chile repositories: 
- `documentation/`: documentation and language instructions.
- `tasks/`: task plans and issues related to the project.

All other repositories are separate child repositories mangaged by this meta repo.
Before you work in a child repository, read its `AGENTS.md` file.
Instructions in a child repository have priority over this file in that
repository.

## How the repositories relate

1. `terraform-google-bigquery-datastream` is a Terraform module. It creates one
   Datastream pipeline from Cloud SQL (PostgreSQL) to BigQuery.
2. `flex-bigquery-terraform` uses the module through a pinned `ref` tag. It also
   creates BigQuery datasets and views.
3. `flex-bigquery-soda` runs Soda checks on the views in `soda_dataset`.

## Cross-repository changes

- Keep each Git commit in one repository.
- Make changes in dependency order. For a module change, follow these steps:
  1. Change `terraform-google-bigquery-datastream`.
  2. Create a new tag in that repository.
  3. Update `ref` in `flex-bigquery-terraform`.
- When you change a view in `soda_dataset`, update the contracts in
  `flex-bigquery-soda`. A changed view can break a check.
- Run Git commands from the child repository folder, not from the workspace
  folder.

## Writing style

Follow `writing-style.md` for all documentation, instructions and explanations.

# Workflow

- Commit to `main` in the top-level meta respository.
- Create a branch before commiting in the child directories
- Do not push commits to upstream unless asked to.
