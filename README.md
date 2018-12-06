# Remote-Hive-Query
> Guide to remotely querying Hive data from a Python Flask web server running in Pivotal Cloud Foundry
 
The ability to connect and utilize a database remotely is a crucial component for lightweight cloud based apps. This repository contains a guide, along with example code, to remotely connect and query data stored in Hive from a python environment in Pivotal Cloud Foundry. In summary, this is done by creating a secure shell (SSH) client in the cloud container and connecting to the edge node Hadoop interface.
 
Furthermore, this short documentation will contain example code for converting the results into a pandas DataFrame and rendering the data in an HTML table using a python Flask web-server.
 
 
![Remote Hive Connection Diagram]()
 
## Import requirements
 
```py
import os
import pandas as pd
from flask import Flask, render_template
import paramiko
```
 
## Usage example
 
Within any flask app route, initialize an SSH Client using paramiko. It's important to configure the client to automatically add a host key when prompted:
 
```py
edge_node = paramiko.SSHClient()
edge_node.set_missing_host_key_policy(paramiko.AutoAddPolicy())
```
 
As the object name implies, the client will be used to SSH into the edge node which we will use to interact with the Hadoop cluster:
 
```py
edge_node.connect('[SERVER ADDRESS]', username='[USERNAME]', password='[PASSWORD]')
```
 
Using exec_command, a beeline command can be submitted using the argument -e followed by a Hive query. The results will be printed to the standard output which we can catch in a Channel File object (edge_out in this example):
 
```py
edge_in, edge_out, edge_err = edge_node.exec_command('beeline -u [HiveServer2 ADDRESS] --silent=true --outputformat=[FORMAT] -e "[QUERY]"')
```
 
The query results are contained in edge_out as bytes which can be read using edge_out.read()
 
```py
results = edge_out.read()
```
 
It's recommended to contain everything within a try, except, and finally clause to ensure that the SSH connection is closed:
 
```py
try:
  edge_node = paramiko.SSHClient()
  edge_node.set_missing_host_key_policy(paramiko.AutoAddPolicy())
  edge_node.connect('[EDGE NODE]', username='[USERNAME]', password='[PASSWORD]')
  edge_in, edge_out, edge_err = edge_node.exec_command('beeline -u [HiveServer2 ADDRESS] --silent=true --outputformat=[FORMAT] -e "[QUERY]"')
  results = edge_out.read()
except:
  "There was an exception..."
finally:
  if edge_node:
    edge_node.close()
```
 
## Inserting query results into a Pandas DataFrame
 
Beeline will return the results of the hive query with lines terminated by '\\\n'. In order to import the results into a pandas DataFrame, the bytes object will need to be converted to a string with lines terminated by '\n':
 
```py
edge_out_str = str(edge_out.read())
edge_out_str_n = "\n".join(edge_out_str.split("\\n"))
```
 
Next, convert the string into a StringIO object and then read that object directly into a Pandas DataFrame:
 
```py
edge_out_csv = StringIO(edge_out_str_n)
df = pd.read_csv(edge_out_csv, sep=",")
```
 
## Implementation into a Flask App
 
Typically, to implement the data returned from a query into the Jinja2 template to be rendered, it's convenient to pass the column names and data as seperate iterables. This can be achieved by converting the pandas DataFrame into a dictionary and appending each key (column name) into a seperate list. The column names and data can then be passed into the render_template function and rendered within the .html code:
 
```py
app = Flask(__name__)
 
#Set the port dynamically with a default of 3000 for local development
port = int(os.getenv('PORT', '3000'))
 
@app.route('/')
def getdata():
  #-----------------------------------------------------------------------------------------------
  #Place code here to send query over SSH, collect results, and convert into a pandas DataFrame df
  #-----------------------------------------------------------------------------------------------
  tabledata = df.to_dict('records')
  column_names = []
  for key, value in tabledata[0].items():
    column_names.append(key)
return render_template('page.html', tabledata=tabledata, column_names=column_names)
```
 
In the template page.html, the table can then be created by iterating through the two objects column_names and tabledata:
 
```html
<table id="table_id" class="display">
    <thead>
        <tr>
                                {% for name in column_names %}
                                    <th>{{ name }}</th>
                                {% endfor %}
        </tr>
    </thead>
    <tbody>
                {% for rows in tabledata %}
        <tr>
                                {% for key, value in rows.items() %}
                                    <td>{{ value }}</td>
                                {% endfor %}
        </tr>
                {% endfor %}
    </tbody>
</table>
```
 
That's it! You can get creative on the front-end by utilizing some powerful client-side javascript libraries such as [DataTables](https://datatables.net/) and [Bokeh](https://bokeh.pydata.org/en/latest/docs/dev_guide/bokehjs.html) to add an array of functionality and visualization.
 
## Meta
 
Sam L. Setegne – (Feel free to reach out if you run into any issues or have questions)
 
[https://github.com/samsetegne](https://github.com/samsetegne)
 
## Contributing
 
1. Fork it (<https://github.com/samsetegne/Remote-Hive-Query/fork>)
2. Create your feature branch (`git checkout -b feature/fooBar`)
3. Commit your changes (`git commit -am 'Add some fooBar'`)
4. Push to the branch (`git push origin feature/fooBar`)
5. Create a new Pull Request
