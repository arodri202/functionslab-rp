# Resource Principle Function for Returning the Instances of the Calling Compartment.

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
  oci-cli
  oci>=2.2.18
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
  Upon success, you should see all of the instances in your compartment appear in your terminal.
