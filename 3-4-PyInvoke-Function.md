# Using OCI Config and Command Line to Invoke an Oracle Function

## Create your Python development Environment

![user input icon](images/userinput.png)
  >1. Verify pip is installed
    ```
    pip --version
    ```
    If it is not, instal pip
    ```
    curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
    ```
  >2. Install Virtualenv
    ```
    pip install virutalenv
    ```
  >3. Create a Virtual Environment and install the OCI Python SDK
    ```
    python3 -m venv .venv
    source .venv/bin/activate
    pip install oci
    ```
  >4. Create a new directory, enter it and create a new .py file
    ```
    mkdir invoke-function
    cd invoke-function
    touch invoke.py
    ```

  In this example we'll show how you can invoke an Oracle Function from a separate Oracle function with only the desired compartment name, application name, the function name, and a request payload by using a Resource Principal Security Signer to get authentication. We will use three API Clients exposed by the [OCI Python SDK](https://oracle-cloud-infrastructure-python-sdk.readthedocs.io/en/latest/index.html).


  >1. [IdentityClient](https://oracle-cloud-infrastructure-python-sdk.readthedocs.io/en/latest/api/identity/client/oci.identity.IdentityClient.html) is an API for managing users, groups, compartments, and policies, we will use it to find the compartment's OCID from its name.

  >2. [FunctionsManagementClient](https://oracle-cloud-infrastructure-python-sdk.readthedocs.io/en/latest/api/functions/client/oci.functions.FunctionsManagementClient.html) - is used for functions lifecycle management operations including creating, updating, and querying applications and functions

  >3. [FunctionsInvokeClient](https://oracle-cloud-infrastructure-python-sdk.readthedocs.io/en/latest/api/functions/client/oci.functions.FunctionsInvokeClient.html#oci.functions.FunctionsInvokeClient) - is used specifically for invoking functions


Writing the Function
------------------
## Open invoke.py
  Import the following packages.

  ![user input icon](images/userinput.png)
  ```python
  import sys
  import os

  import oci.functions
  import oci.config
  from oci import identity
  from oci import pagination
  ```

## The main method
  This is what is called when the function is invoked via the command line.

  ![user input icon](images/userinput.png)
  ```python
  if __name__ == "__main__":
      main()
  ```
  Now we need to define `main()`
  ```python
  if len(sys.argv) != 5:
        raise Exception("usage: python invoke_function.py"
                        " <compartment-name> <app-name> "
                        "<function-name> <request payload>")

    compartment_name = sys.argv[1]
    app_name = sys.argv[2]
    fn_name = sys.argv[3]

    cfg = oci.config.from_file(
        file_location=os.getenv(
            "OCI_CONFIG_PATH", oci.config.DEFAULT_LOCATION),
        profile_name=os.getenv(
            "OCI_CONFIG_PROFILE", oci.config.DEFAULT_PROFILE)
    )

    functions_client = oci.functions.FunctionsManagementClient(config=cfg)
    oci.config.validate_config(cfg)

    compartment = get_compartment(cfg, compartment_name)

    app = get_app(functions_client, app_name, compartment)

    fn = get_function(functions_client, app, fn_name)

    invoke_client = oci.functions.FunctionsInvokeClient(
        cfg, service_endpoint=fn.invoke_endpoint)

    resp = invoke_client.invoke_function(fn.id, invoke_function_body=sys.argv[4])
    print(resp.data.text)
  ```

## The get_compartment method
  Copy and paste this method into your file. This method identifies the compartment ID by a given name within the particular tenancy

    :param oci_cfg: OCI auth config

    :param compartment_name: OCI tenancy compartment name

    :return: OCI tenancy compartment

    ![user input icon](images/userinput.png)
  ```python
  def get_compartment(oci_cfg, compartment_name):
    identity_client = identity.IdentityClient(config=oci_cfg)
    result = pagination.list_call_get_all_results(
        identity_client.list_compartments,
        oci_cfg["tenancy"],
        compartment_id_in_subtree=True,
        access_level="ACCESSIBLE",
    )
    for c in result.data:
        if compartment_name == c.name:
            print(type(c))
            return c
  ```

## The get_app method
  Copy and paste this method into your file. This method identifies an OCI Function app object by its name

  parameters:
    functions_client: OCI Functions FunctionsManagementClient

    app_name: OCI Functions app name

    compartment: OCI tenancy compartment

  return: OCI Functions app

  ![user input icon](images/userinput.png)
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
  Copy and paste this method into your file. This method identifies an OCI function object by its name

  parameters:

    functions_client: OCI Functions FunctionsManagementClient

    app: OCI Functions app

    function_name: OCI Functions function name

  return: OCI Functions function
  ```python

  ![user input icon](images/userinput.png)
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

## Invoke the function using CLI

  ![user input icon](images/userinput.png)
  ```
  python invoke_function.py <compartment-name> <app-name> <function-name> <request payload>
  ```

  e.g.

  ```
  python invoke.py workshop ws<NNN>app node-fn '{"name":"DOAG"}'
  ```
  Upon success, you should see the expected output of your function.

  ```
  {"message":"Hello DOAG"}
  ```
