# biochef-infra

The BioChef repositories as Git submodules, each pinned to a commit on its default branch.

| Submodule | Repository | Responsibility |
|---|---|---|
| `biochef-app` | [ieeta-pt/Biochef](https://github.com/ieeta-pt/Biochef) | The browser workflow editor ([biochef.app](https://biochef.app)), which runs tools as WebAssembly. |
| `biochef-agent` | [ieeta-pt/biochef-agent](https://github.com/ieeta-pt/biochef-agent) | An execution endpoint that runs workflows as Snakemake next to the data (TRE, HPC, institutional server). |
| `biochef-coordinator` | [ieeta-pt/biochef-coordinator](https://github.com/ieeta-pt/biochef-coordinator) | The central coordination hub. Studies, per-node workflow approvals and jobs, which agents fetch by polling. |
| `biochef-hub` | [ieeta-pt/biochef-hub](https://github.com/ieeta-pt/biochef-hub) | The CI that validates, builds, tests, signs and publishes recipes to the BioChef Registry. |
| `biochef-recipes` | [ieeta-pt/biochef-recipes](https://github.com/ieeta-pt/biochef-recipes) | The tool recipes (`biochef.yaml` plus tiny tests) that the hub builds into signed bundles. |

## Clone and update

```sh
git clone --recurse-submodules git@github.com:ieeta-pt/biochef-infra.git
git submodule update --remote   # move every submodule to the latest commit on its default branch
```
