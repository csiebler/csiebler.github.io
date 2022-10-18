---
title: "Using Cognitive Services Read API with data secured by VNET"
date: 2022-22-02T19:30:53+02:00
---

This post discusses how Cognitive Services can be used to process data that is securely stored behind a VNET. This allows to improve security further when processing sensitive data that is stored in Storage Accounts. In this post, we’ll look into using Read API (from the Azure Computer Vision API) to analyze documents that are heavily protected using networking rules.

This post also applies to other Cognitive Services, such as for example [Form Recognizer](https://azure.microsoft.com/en-us/services/form-recognizer/) or [Speech API](https://azure.microsoft.com/en-us/services/cognitive-services/speech-services/).

## Data behind VNET – what does it mean?

In Azure, many users protect their data using a set of security perimeters. While these are typically not all measurements users take to store their data securely, those are the most common ones that most people use:

* Authentication layer – requirement to authenticate at the storage layer, e.g., via a access key or based on an identity (could be a real user or a system managed identity)
* Authorization layer – what is the identity allowed to do with the data (just read certain folders, write data, delete, etc.)
* Networking layer – from where is the identity allowed to access the data (from the internet, from a range of IP addresses, only from within a VNET, from “nowhere”)
For example, your Networking settings on your Storage Account might look like this:

![Storage Account](/images/storage_account.png "Typical Networking settings on a Storage Account")

In this case, the data can only be access, when the access request originates from within subnet default inside the VNET `vnet-test`.

Using these security measurements makes it incredibly hard/potentially impossible to have an attacker access the data. But this also creates a problem: what if an Azure service, like for example a Cognitive Service needs to access this data? The service might be authenticated (e.g., using a SAS URL), but from a networking perspective, the service is obviously not coming from within your VNET:

![No storage account access allowed](/images/vnet_storage_access_denied.png)

In the drawing above the Read API is:

* Authenticated to access the Storage Account and read the data (using the SAS URL from the request)
* Blocked by the networking rules of the Storage account

If we would remove the networking rule on the storage account, the data access would obviously be allowed:

![Storage account access allowed](/images/no_vnet_storage_access_allowed.png)

However, this is obviously not the desired setup as the Storage Account might hold sensitive data.

### Desired state

Obviously, we want to make sure that we do everything possible to protect our data as much as we can. Furthermore, it would also be even more secure if we did not have to use a SAS URL at all. Sure, this URL might be short lived, but why not get rid of it if we can? But most importantly, we want to make sure we can use our Cognitive Services, despite the data being sitting behind a VNET.

## The solution – Managed Identity and Resource Service Instances to the rescue!

The solution to make this scenario work requires two components:
* [Managed Identities](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)
* [Resource Service Instances](https://docs.microsoft.com/en-us/azure/storage/common/storage-network-security?tabs=azure-portal#grant-access-from-azure-resource-instances-preview)

### Managed Identities

Most Azure services can "assume" a **Managed Identity**. A Managed Identity is a system user that is tied to that specific Azure resource. Since it is a “user”, the identity lives in your Azure Active Directory. That means we can assign that identity other privileges, such as access to storage or other Azure services. This means we can firstly assign our Cognitive Service a Managed Identity. Secondly, we can allow that identity to be able to read/write to our storage account using IAM. As a result, the Cognitive Service can access data without the need for access keys. Instead, it automatically requests an OAuth token from AAD and uses it to authenticate to the storage.

### Resource Service Instances

However, the networking layer will still block the request and this is where **Resource Service Instances** come into play. With this feature we can specify a list of Azure services that are allowed to connect to the storage account regardless of its networking settings. This means, we can explicitly permit our Cognitive Service resource to “tunnel through” the firewall. Is this a security issue? No, because it is only allowed for the identity of the Cognitive Service itself. And as this identity is a system user, it is automatically secured.

### Overall Architecture

Putting both together, we get to this:

![Read API can access the storage account using its Managed Identity](/images/vnet_storage_access_allowed.png)

In this case:

* Read API uses its Managed Identity to authenticate at the Storage Account (no SAS URL required!)
* The Resource Service Instance configuration on the Storage Account allows the Managed Identity to "get through" the VNET-firewall

## Step-by-step guide

To try this out, first create a new Computer Vision API (this includes the Read API):

![Create a new Computer Vision API](/images/aad_computer_vision_create.png)

During the creation, make sure to enable **Managed Identity**. You can also always later enable it under the **Identity** tab:

![Managed Identity on Computer Vision API](/images/computer_vision_managed_identity.png)

Next, click **Azure Role Assignments** on the same screen and select **Add role assignment**. Then, assign the **Storage Blob Data Reader** role to your **Storage Account**. Once done, you could send plain storage URLS without SAS tokens to Read API and it could read the data.

Next, navigate to your Storage Account, select **Networking** and check the network settings. In our example here, we only allow access from **selected networks**. Ironically, we did not select any VNET, so the data can’t be accessed from anywhere, including Cognitive Services. However, we’ll add the **Cognitive Services Resource type** and then name of our **Cognitive Service instance**. This means our Cognitive Service can tunnel through this super-restrictive networking setting!

![Allowing our Cognitive Service resource to tunnel through the firewall](/images/networking_settings_storage.png)

Don't forget to hit the save button.

### Testing the whole setup

Once done, we can fire a few REST API calls to send a document to the Read API:

```http
# @name read_document
POST https://computer-vision-demo124.cognitiveservices.azure.com/vision/v3.2/read/analyze
     ?language=en
     &pages=1
     &readingOrder=natural
Content-Type: application/json
Ocp-Apim-Subscription-Key: <secret key>

{
    "url": "https://dgjt35hksss.blob.core.windows.net/data/description.png"
}

##### This request queries the status of the Read API operation

# @name get_results
GET {{read_document.response.headers.Operation-Location}}
Content-Type: application/json
Ocp-Apim-Subscription-Key: <secret key>
```

The result looks successful:

```json
{
  "status": "succeeded",
  "createdDateTime": "2022-02-22T12:52:59Z",
  "lastUpdatedDateTime": "2022-02-22T12:53:00Z",
  "analyzeResult": {
    "version": "3.2.0",
    "modelVersion": "2021-04-12",
    "readResults": [
      {
        "page": 1,
        "angle": -0.7095,
...
```

Despite us just sending a regular URL (`https://dgjt35hksss.blob.core.windows.net/data/description.png`), the Read API can access the data:

* Authenticated through its Managed Identity and
* allowed through the firewall by the Resource Instance configuration

If we remove the Resource instance definition, we would get the following message, as the URL would return a 403 error to the Cognitive Service:

```json
{
  "error": {
    "code": "InvalidImageURL",
    "message": "Failed to download the image from the submitted URL. The URL may either be invalid or the server hosting the image is experiencing some technical difficulties."
  }
}
```

Great, looks like it is working fine now!

By the way, we can entirely avoid the secret to call the Cognitive Service itself. If you are interested, have a look at this blog post: Azure Active Directory (AAD) authentication for Azure Cognitive Services.

## Summary

Many users want to protect their sensitive data using as many measurements there are available on the Azure platform:

* Firstly, this will be using authentication and authorization for accessing data on storage
* Secondly, this will include networking rules to limit from where data can be accessed

This creates a unique challenge for accessing this VNET-protected data using Cognitive Services. However, combing the usage of **Managed Identities** and **Resource Service Instances** solve the problem. They enable users to keep their data well protected, but still allows them to process it by Cognitive Services.