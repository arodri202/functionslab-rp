# Using Instance Principals to Invoke an Oracle Function from an OCI Compute Instance Virtual Machine

## Reuse your first app

We will reuse the app you created in the [Hello World - Node](3-2-NodeHello.md) example. We will add the Python function to the same app.

  Get the python boilerplate by running:

  ![user input icon](images/userinput.png)
  ```
  fn init --runtime python <function-name>
  ```
  e.g.
  ```
  fn init --runtime python invoke-function
  ```

  In this example we'll show how you can invoke an Oracle Function from a separate Oracle function with only the desired compartment name, application name, the function name, and a request payload by using a Resource Principal Security Signer to get authentication. We will use three API Clients exposed by the [OCI Python SDK](https://oracle-cloud-infrastructure-python-sdk.readthedocs.io/en/latest/index.html).


  1. [IdentityClient](https://oracle-cloud-infrastructure-python-sdk.readthedocs.io/en/latest/api/identity/client/oci.identity.IdentityClient.html) is an API for managing users, groups, compartments, and policies, we will use it to find the compartment's OCID from its name.

  2. [FunctionsManagementClient](https://oracle-cloud-infrastructure-python-sdk.readthedocs.io/en/latest/api/functions/client/oci.functions.FunctionsManagementClient.html) - is used for functions lifecycle management operations including creating, updating, and querying applications and functions

  3. [FunctionsInvokeClient](https://oracle-cloud-infrastructure-python-sdk.readthedocs.io/en/latest/api/functions/client/oci.functions.FunctionsInvokeClient.html#oci.functions.FunctionsInvokeClient) - is used specifically for invoking functions


Writing the Function
------------------
## Requirements
  Update your requirements.txt file to contain the following:

  ![user input icon](images/userinput.png)
  ```
  fdk
  oci-cli
  oci>=2.2.18
  ```

## Open func.py
  Update the imports so that you contain the following.

  ![user input icon](images/userinput.png)
  ```python
  import sys
  import io
  import json
  import oci.auth
  import oci.functions
  from fdk import response

  from oci import identity
  from oci import pagination
  ```

## The Handler method
  This is what is called when the function is invoked by Oracle Functions, delete what is given from the boilerplate and update it to contain the following:

  ![user input icon](images/userinput.png)
  ```python
  def handler(ctx, data: io.BytesIO=None):
    compartment_name = ""
    app_name = ""
    fn_name = ""
    fn_body = ""
    try:
        body = json.loads(data.getvalue())
        compartment_name = str(body["compartment"])
        app_name = str(body["app"])
        fn_name = str(body["function"])
        fn_body = str(body["payload"])

    except Exception as e:
        error = 'Input a JSON object with keys: compartment, app, function, payload. ' + str(e)
        raise Exception(error)

    resp = do(compartment_name, app_name, fn_name, fn_body)

    return response.Response(
        ctx, response_data=json.dumps(resp),
        headers={"Content-Type": "application/json"}
    )
  ```

## The do method
  Create the following method.

  ![user input icon](images/userinput.png)
  ```python
  def do(compartment_name, app_name, fn_name, fn_body):

    signer = oci.auth.signers.get_resource_principals_signer()

    tenancy = signer.tenancy_id

    functions_client = oci.functions.FunctionsManagementClient(config={}, signer=signer)

    compartment = get_compartment(signer=signer, tenancy=tenancy, compartment_name=compartment_name)

    app = get_app(functions_client, app_name, compartment)

    fn = get_function(functions_client, app, fn_name)

    invoke_client = oci.functions.FunctionsInvokeClient(config={}, signer=signer, service_endpoint=fn.invoke_endpoint)
    if fn_body != None:
        resp = invoke_client.invoke_function(fn.id, invoke_function_body=fn_body)
        print(fn_body, file=sys.stderr)
    else:
        resp = invoke_client.invoke_function(fn.id)

    return resp.data.text
  ```

## get_compartment
  Identifies compartment ID by its name within the particular tenancy
  parameters:
    signer: OCI Resource Principal Signer,
    tenancy: OCI tenancy name
    compartment_name: OCI tenancy compartment name
  return: OCI tenancy compartment
  ```python
  def get_compartment(signer, tenancy, compartment_name):
    identity_client = identity.IdentityClient(config={}, signer=signer)
    result = pagination.list_call_get_all_results(
        identity_client.list_compartments,
        tenancy,
        compartment_id_in_subtree=True,
        access_level="ACCESSIBLE",
    )
    for c in result.data:
        if compartment_name == c.name:
            print(type(c))
            return c

    raise Exception("compartment not found")
  ```

## get_app
  Identifies app object by its name
  parameters:
    functions_client: OCI Functions FunctionsManagementClient
    app_name: OCI Functions app name
    compartment: OCI tenancy compartment
  return: OCI Functions app
  ```python
  def get_app(functions_client, app_name, compartment):
    result = pagination.list_call_get_all_results(
        functions_client.list_applications,
        compartment.id
    )
    for app in result.data:
        if app_name == app.display_name:
            print(type(app))
            print(app.display_name)
            return app

    raise Exception("app not found")
    ```

## get_function
  Identifies function object by its name
  parameters:
    functions_client: OCI Functions FunctionsManagementClient
    app: OCI Functions app
    function_name: OCI Functions function name
  return: OCI Functions function
  ```python
  def get_function(functions_client, app, function_name):
    result = pagination.list_call_get_all_results(
        functions_client.list_functions,
        app.id
    )
    for fn in result.data:
        if function_name == fn.display_name:
            print(type(fn))
            print(fn.display_name)
            return fn

    raise Exception("function not found")
  ```

## Deploy the function

  ![user input icon](images/userinput.png)
  >```
  >fn -v deploy --app ws<NNN>app
  >```

## Invoke the function using CLI

  ![user input icon](images/userinput.png)
  ```
  echo -n '{"compartment": "<compartment-name>", "app": "<app name>", "function": "<function name>", "payload": "<function payload>"}' | fn invoke ws<NNN>app <your function name>
  ```

  e.g.

  ```
  echo -n '{"compartment": "workshop", "app": "ws<NNN>", "function": "node-fn", "payload": ""}' | fn invoke ws<NNN>app invoke-function
  ```
  Upon success, you should see the expected output of your function.
