# UpsertDatabricksNotebooks
Automatically copy a notebook from specified folder to another one

```yaml
name: UpsertDatabricksNotebook

on:
  push:
    branches: [ master ]
    paths: 
      - 'notebooks/Shared/development/**'
  pull_request:
    branches: [ master ]
env:
  DEV_FOLDER: '/Shared/development'
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
