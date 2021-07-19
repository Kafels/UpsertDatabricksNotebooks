# Upsert Databricks Notebooks

# Requisites
- Connect Databricks to GitHub. Step by step [here](https://docs.databricks.com/repos.html)
- Create a GitHub Action. See an example [here](https://docs.github.com/en/actions/quickstart)
- Configure GitHub Secrets to hide sensitive credentials
- Copy the provided YAML below and change `on` and `env` sections to fit your environment

---

### GitHub Secrets
Your repository must have these two variables:
- DATABRICKS_HOST (should begin with https://)
- DATABRICKS_TOKEN ([Generate a personal access token](https://docs.databricks.com/dev-tools/api/latest/authentication.html#generate-a-personal-access-token))

Example:
![secrets](https://user-images.githubusercontent.com/17178349/121604485-f9952f00-ca20-11eb-848f-96b862e4a442.png)

---

### YAML
```yaml
name: UpsertDatabricksNotebook

on:
  push:
    # Branch to trigger this pipeline
    branches: [ master ]
    paths:
      # The directory to watch commits
      - 'notebooks/Shared/development/**'
  pull_request:
    # Also trigger when executing PR
    branches: [ master ]
env:
  # Databricks folder containing all development scripts
  DEV_FOLDER: '/Shared/development'
  # Databricks folder where all development code must be uploaded 
  PRD_FOLDER: '/Shared/production'
jobs:
  publish_on_production:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Install python dependencies
        run: |
          echo "Installing pip dependencies"
          python3 -m pip install --upgrade pip setuptools wheel --user
          echo "Installing databricks-cli"
          python3 -m pip install databricks-cli==0.12.0 --user
          
      - name: Setup databricks environment
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
        run: |  
          DATABRICKS_CONFIG_FILE="$HOME/.databrickscfg"
          
          echo "[DEFAULT]" > $DATABRICKS_CONFIG_FILE
          echo "host = $DATABRICKS_HOST" >> $DATABRICKS_CONFIG_FILE
          echo "token  = $DATABRICKS_TOKEN" >> $DATABRICKS_CONFIG_FILE
          
          echo "$HOME/.local/bin" >> $GITHUB_PATH
          echo "DATABRICKS_HOST=$DATABRICKS_HOST" >> $GITHUB_ENV
          echo "DATABRICKS_TOKEN=$DATABRICKS_TOKEN" >> $GITHUB_ENV
          echo "DATABRICKS_CONFIG_FILE=$DATABRICKS_CONFIG_FILE" >> $GITHUB_ENV
      
      - name: Lookup files
        id: files
        uses: jitterbit/get-changed-files@v1
      
      - name: Publish
        run: |
          for modified_file in ${{ steps.files.outputs.all }}; do
            if [[ $modified_file == *$DEV_FOLDER* ]]; then
              notebook=$(echo $modified_file | sed -r 's/://g')
              notebook_path=$(echo $notebook | cut -d'/'  -f4-)
              dev_file_path="$DEV_FOLDER/$notebook_path"
              local_file_path="$GITHUB_WORKSPACE/notebooks/$dev_file_path"
              databricks_file_path="$PRD_FOLDER/$notebook_path"
              databricks_file_path=$(echo $databricks_file_path | cut -d'.'  -f1)
              if [[ -e "$local_file_path" ]]; then
                echo "Uploading:  ${dev_file_path} -> ${databricks_file_path}"
                databricks workspace mkdirs $(dirname $databricks_file_path)
                databricks workspace import -l PYTHON --overwrite $local_file_path $databricks_file_path
              else
                echo "Removing: ${databricks_file_path}"
                databricks workspace rm $databricks_file_path
              fi
            fi
          done
```
