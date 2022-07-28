# dbt_to_tabcatalog
script for automating column name/descriptions to tableau catalog table descriptions

### Authenticate to dbt
### Get Account ID
```py
def get_account_id(token):
    
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
```py
def get_jobs(token)
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
```py
def get_models_information(token, job_id):

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
### Authenticate to Tableau
```py
 def authenticate_tableau(url_name,PAT,site_name, token_name):
    url = "https://" + url_name + " /api/3.13/auth/signin"

    payload = json.dumps({
      "credentials": {
        "personalAccessTokenName": token_name,
        "personalAccessTokenSecret": PAT,
        "site": {
          "contentUrl": site_name
        }
      }
    })
    headers = {
      'Content-Type': 'application/json'
    }

    response = requests.request("POST", url, headers=headers, data=payload)
    response = response.text
    token_string = response.split('token="',2)
    token_string_split1 = token_string[1].split('"',1)
    token = token_string_split1[0]
    
    site_id = token_string[2].split('site id="',1)
    site_id = site_id.split('"')
    site_id = site_id[0]
    
    req_strings=[token,site_id]
    return(req_strings)
```
### Get list of tables from a database
```py
def get_table_luids(url_name, database_type, database_name, token)
mdapi_query = '''
query get_databases {
  databases (filter: {connectionType: " +database_type+ ", name:"''' + database_name + '''"}){
 name
 id
 tables{
     name
     id
     luid
 
 }

 
}
}
'''
auth_headers = auth_headers = {'accept': 'application/json','content-type': 'application/json','x-tableau-auth': token}
metadata_query = requests.post('https://'+ts_url + '/api/metadata/graphql', headers = auth_headers, verify=True, json = {"query": mdapi_query})
mdapi_result = json.loads(metadata_query.text)

k=0
table_luid_list = []
while k < len(mdapi_result['data']['databases'][0]['tables']):
    print(mdapi_result['data']['databases'][0]['tables'][k]['luid'])
    table_luid_list.append(mdapi_result['data']['databases'][0]['tables'][k]['luid'])
    k = k+1
    
table_dictionary = {"table_name":table_name_list, "table_luid":table_luid_list}
tableau_tables_info = pd.DataFrame(table_dictionary, columns=['table_name','table_luid'])

return(tableau_tables_info)
```

### Get table ID
```py
def get_table_id(url_name, table_name, site_id, token):
    get_tables_url = "https://"+url_name+"/api/3.13/sites/"+site_id+"/tables"

    payload = ""
    headers = {
      'X-Tableau-Auth': token
    }

    table_response = requests.request("GET", get_tables_url, headers=headers, data=payload)
    table_id_string = table_response.text.split('name="'+table_name+'"',1)
    table_id_string_split1 = table_id_string[0].split('<table id="')
    table_id_string_split2 = table_id_string_split1[-1]
    table_id_string_split3 = table_id_string_split2[0:-1]
    table_id_string_split4 = table_id_string_split2[0:-2]
    return(table_id_string_split4)
```

### Get column names and column IDs
```py
def get_list_of_columns(url_name,table_id,site_id,token):
    get_columns_url = "https://"+url_name+"/api/3.13/sites/"+site_id+"/tables/"+table_id+'/columns'

    payload = ""
    headers = {
      'X-Tableau-Auth': token
    }

    columns_response = requests.request("GET", get_columns_url, headers=headers, data=payload)
    columns_response_split = columns_response.text.split('" name="')
    i = 0
    column_names_list = []
    column_ids_list = []
    while i < len(columns_response_split):
        if i%2 == 0: 
            #even number, grab the column ID
            column_id = columns_response_split[i].split('column id="', 1)
            column_id = column_id[1][0:-2]
            column_ids_list.append(column_id)
        if i%2 == 1:
            #odd number, this is to grab the name
            column_name = columns_response_split[i].split('"',1)
            column_name = column_name[0]
            column_names_list.append(column_name)
        i = i+1
    
    column_dictionary = {"column_name":column_names_list, "column_id":column_ids_list}
    tableau_column_info = pd.DataFrame(column_dictionary, columns=['column_name','column_id'])
    return(tableau_column_info)
 ```
### Merge column descriptions and IDs
```py
def add_comments_to_tab_table(tableau_columns, dbt_columns):
    join_result = tableau_columns.merge(dbt_columns, how='inner',left_on="column_name",right_on='name')

    return(join_result)

joined_tables = add_comments_to_tab_table(fct_order_items, columns_table)
print(joined_tables)
```

### Publish descriptions to columns
```py
def publish_description_to_column(url_name,site_id,table_id,column_id, description_text,token):
    column_description_url = "https://"+url_name+"/api/3.13/sites/"+site_id+"/tables/" + table_id + "/columns/" + column_id

    payload = "<tsRequest>\n  <column description=\"" + description_text +" \">\n  </column>\n</tsRequest>"
    headers = {
        'X-Tableau-Auth': token,
      'Content-Type': 'text/plain'
    }

    column_description_response = requests.request("PUT", column_description_url, headers=headers, data=payload)

    column_description_response_code = column_description_response.text
    return(column_description_response_code)
 ```
### Publish description to table

```py
def publish_description_to_table(url_name,site_id,table_id, description_text,token):
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
