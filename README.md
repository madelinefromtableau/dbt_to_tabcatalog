# dbt_to_tabcatalog
script for automating column name/descriptions to tableau catalog table descriptions

### Authenticate to dbt
### Get Account ID

```def get_account_id(token):
    
    url = "https://cloud.getdbt.com/api/v2/accounts/"

    payload={}
    headers = {
      'Content-Type': 'appication/json',
      'Authorization': 'Token '+ token
    }

    response = requests.request("GET", url, headers=headers, data=payload)
    #print(response.text)
    response_json = json.loads(response.text)
    account_id = response_json['data'][0]['id']
    return(account_id)
```

### Get projects
### Get jobs based on a project
```def get_jobs(token)
    url = "https://cloud.getdbt.com/api/v2/accounts/60964/jobs"

    payload={}
    headers = {
      'Content-Type': 'appication/json',
      'Authorization': 'Token ccd277aedc74569a97db555667f3785f33a73f13'
    }

    response = requests.request("GET", url, headers=headers, data=payload)
    response_json = json.loads(response.text)
    job_dataframe = pd.json_normalize(response_json['data'])
    name_plus_job_id = job_dataframe[['id','name']]
    return(name_plus_job_id)
```
    
### Get JobID
### Get information about a job
### create table of models/executecompletedat/uniqueId
```def get_models_information(token, job_id):

    url = "https://metadata.cloud.getdbt.com/graphql"
    job_id = '102781'
    payload="{\"query\":\"{\\n  models(jobId: " + job_id + ") {\\n    uniqueId\\n    executionTime\\n    status\\n    executeCompletedAt\\n    dependsOn\\n    database\\n    schema\\n    columns {\\n        name\\n        description\\n    }\\n\\n  }\\n}\",\"variables\":{}}"
    headers = {
      'Authorization': 'Token ' + token,
      'Content-Type': 'application/json'
    }

    response = requests.request("POST", url, headers=headers, data=payload)
    response_json = json.loads(response.text)
    models_table = pd.json_normalize(response_json['data']['models'])
    models_table = models_table[['uniqueId', 'executeCompletedAt', 'database']]
    return(models_table)
```
    
### create table of column names and descriptions with unique Id
### Authenticate to tableau
### Get list of databases
### filter to one database get tables
### Get column names and column IDs
### Merge column descriptions and IDs
### Publish descriptions to columns

### Publish description to table

```def publish_description_to_table(url_name,site_id,table_id, description_text,token):
    column_description_url = "https://"+url_name+"/api/3.13/sites/"+site_id+"/tables/" + table_id

    payload = "<tsRequest>\n  <description=\"" + description_text +" \">\n  </table>\n</tsRequest>"
    headers = {
        'X-Tableau-Auth': token,
      'Content-Type': 'text/plain'
    }

    table_description_response = requests.request("PUT", column_description_url, headers=headers, data=payload)

    table_description_response_code = table_description_response.text
    return
```
