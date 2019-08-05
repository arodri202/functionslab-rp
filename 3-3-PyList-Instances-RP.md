# Resource Principle Function for Returning the Instances of an OCI Compartment.

## Reuse your first app

We will reuse the app you created in the [Hello World - Node](3-2-NodeHello.md) example. We will add the Python function to the same app.

  Get the python boilerplate by running:

  ![user input icon](images/userinput.png)
  ```
  fn init --runtime python <function-name>
  ```
  e.g.
  ```
  fn init --runtime python list-instances
  ```

Writing the Function
------------------
## Requirements
  Update your requirements.txt file to contain the following:

  ![user input icon](images/userinput.png)
  ```
  fdk
  oci
  ```

## Open func.py
  Update the imports so that you contain the following.

  ![user input icon](images/userinput.png)
  ```python
  import io
  import json
  from fdk import response

  import oci
  ```

## The Handler method
  This is what is called when the function is invoked by Oracle Functions, delete what is given from the boilerplate and update it to contain the following:

  ![user input icon](images/userinput.png)
  ```python
  def handler(ctx, data: io.BytesIO=None):
      signer = oci.auth.signers.get_resource_principals_signer()
      resp = do(signer)
      return response.Response(
          ctx, response_data=json.dumps(resp),
          headers={"Content-Type": "application/json"}
      )
  ```

## The do method
  Create the following method.

  ![user input icon](images/userinput.png)
  ```python
  def do(signer):
  ```
  This is where we'll put the bulk of our code that will connect to OCI and return the list of compartments in our tenancy.

  ![user input icon](images/userinput.png)
  ```python
      # List instances (in IAD) --------------------------------------------------------------------------------
      client = oci.core.ComputeClient(config={}, signer=signer)
      # Use this API to manage resources such as virtual cloud networks (VCNs), compute instances, and block storage volumes.
      try:
          inst = client.list_instances(signer.compartment_id)

          inst = [[i.id, i.display_name] for i in inst.data]
      except Exception as e:
          inst = str(e)

      resp = {
               "instances": inst,
              }

      return resp
  ```
  Here we are creating a [ComputeClient](https://oracle-cloud-infrastructure-python-sdk.readthedocs.io/en/latest/api/core/client/oci.core.ComputeClient.html) from the [OCI Python SDK](https://oracle-cloud-infrastructure-python-sdk.readthedocs.io/en/latest/index.html), which allows us to connect to OCI with the resource principal's signer data, since resource principal is already pre-configured we do not need to pass in a valid config dictionary, we are now able to make a call to identity services for information on our compartments.

## Deploy the function

  ![user input icon](images/userinput.png)
  >```
  >fn -v deploy --app ws<NNN>app
  >```

## Invoke the function using CLI

  ![user input icon](images/userinput.png)
  ```
  fn invoke ws<NNN>app <your function name>
  ```

  e.g.

  ```
  fn invoke ws<NNN>app list-instances
  ```
  You should see an error that looks something like this!
  ```
  {"instances": "{'opc-request-id': 'E375A6...7ACD', 'code': 'NotAuthorizedOrNotFound', 'message': 'Authorization failed or requested resource not found.', 'status': 404}"}
  ```
  That's because you should not have any instances right now!

## Creating an Instance
  Log in to https://console.us-ashburn-1.oraclecloud.com/

  1. Click the Dropdown in the upper left corner, find Compute > Instances.

  2. Click Create Instance, and following the on-screen instructions
    > Note: All of your compartments should be workshop

  Upon success, you should see all of the instances in your compartment appear in your terminal.

  Invoke your function again and you should see your newly-created instance!

## Creating Function Configuration
  Currently, your function is looking for instances in the compartment that holds your deployed function. But what if we held all of our instances in a separate compartment? The following steps will show you how to create configuration for your function to be used during run time.

  There are currently 3 ways you can create Function Configuration.
  1. Fn CLI
  2. Functions page on Oracle Console
  3. Modifying the func.yaml file

  In this workshop we will be changing the func.yaml file and verifying that the correct configuration has been created via the Oracle Console.

  Open your func.yaml file and add the following lines to the bottom.
  ```
  config:
    OCI_COMPARTMENT_ID: <TO BE FILLED>
  ```
  Now, navigate through the Oracle Console and select a compartment that does not correspond to your `ws<NNN>` compartment, copy the Compartment OCID and paste it.

## Reading Function Configuration
  Open your `func.py` file, add the following import and global variable.
  ```python
  import os

  OCI_COMPARTMENT_ID = os.environ.get("OCI_COMPARTMENT_ID")
  ```
  Now go to your `do()` method and look for the line:
  ```python
  inst = client.list_instances(signer.compartment_id)
  ```
  Replace `signer.compartment_id` with your new global variable `OCI_COMPARTMENT_ID`

## Redeploy and Verify
  Redeploy your function.
  >```
  >fn -v deploy --app ws<NNN>app
  >```

  To verify that you have the correct configuration. Go to the Oracle Console.
  1. Click the Dropdown menu and find Developer Services > Functions
  2. Find your application `ws<NNN>`
  3. Find your instance function `list-instances`
  4. Under resources on the left side of your screen, click Configuration.
  5. You should see OCI_COMPARTMENT_ID and the value you passed in to your yaml file

  Now invoke your function and you should see the instances of the compartment you passed in!
