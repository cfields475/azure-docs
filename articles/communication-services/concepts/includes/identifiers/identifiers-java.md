---
title: include file
description: include file
services: azure-communication-services
author: DominikMe
manager: RezaJooyandeh

ms.service: azure-communication-services
ms.subservice: azure-communication-services
ms.date: 08/30/2022
ms.topic: include
ms.custom: include file
ms.author: domessin
---

### Communication user

The `CommunicationUserIdentifier` represents a user identity that was created using the [Identity SDK or REST API](../../../quickstarts/identity/access-tokens.md). It's the only identifier used if your application doesn't use Microsoft Teams interoperability or Telephony features.


#### Basic usage

```java
// at some point you will have created a new user identity in your trusted service
CommunicationUserIdentifier newUser = identityClient.CreateUser();

// and then send newUser.getId() down to your client application
// where you can again create an identifier for the user
var sameUser = new CommunicationUserIdentifier(newUserId);
```

#### API reference

[CommunicationUserIdentifier](/java/api/com.azure.communication.common.communicationuseridentifier)

### Microsoft Teams user

The `MicrosoftTeamsUserIdentifier` represents a Teams user with its Azure AD user object ID. You can retrieve the Azure AD user object ID via the [Microsoft Graph REST API /users](/graph/api/user-get) endpoint from the `id` property in the response. For more information on how to work with Microsoft Graph, try the [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer?request=users%2F%7Buser-mail%7D&method=GET&version=v1.0&GraphUrl=https://graph.microsoft.com) and look into the [Graph SDK](/graph/sdks/sdks-overview). Alternatively, you can find the ID as the `oid` claim in an [Azure AD ID token](../../../../active-directory/develop/id-tokens.md#payload-claims) or [Azure AD access token](../../../../active-directory/develop/access-tokens.md#payload-claims) after your user has signed in and acquired a token.

#### Basic usage

```java
// get the Teams user's ID from Graph APIs if only the email is known
var user = await graphClient.users("bob@contoso.com")
    .buildRequest()
    .get();

// create an identifier
var teamsUser = new MicrosoftTeamsUserIdentifier(user.id);

// if you're not operating in the public cloud, you must also set the right Cloud type.
var gcchTeamsUser = new MicrosoftTeamsUserIdentifier(userId).setCloudEnvironment(CommunicationCloudEnvironment.GCCH);
```

#### API reference

[MicrosoftTeamsUserIdentifier](/java/api/com.azure.communication.common.microsoftteamsuseridentifier)

### Phone number

The `PhoneNumberIdentifier` represents a phone number. The service assumes that phone numbers are formatted in E.164 format.

#### Basic usage

```java
// create an identifier
var phoneNumber = new PhoneNumberIdentifier("+112345556789");
```

#### API reference

[PhoneNumberIdentifier](/java/api/com.azure.communication.common.phonenumberidentifier)

### Unknown

The `UnknownIdentifier` exists for future-proofing and you might encounter it when you are on an old version of the SDK and a new identifier type has been introduced recently. Any unknown identifier from the service will be deserialized to the `UnknownIdentifier` in the SDK.

#### Basic usage

```java
// create an identifier
var unknown = new UnknownIdentifier("a raw id that originated in the service");
```

#### API reference

[UnknownIdentifier](/java/api/com.azure.communication.common.unknownidentifier)

### How to handle the `CommunicationIdentifier` base class

While you construct identifiers for a concrete type that you pass *into* the SDK, the SDK returns the abstract `CommunicationIdentifier`. You can down-cast back to a concrete type:

```java
if (communicationIdentifier instanceof CommunicationUserIdentifier) {
    System.out.println("Communication user: " + ((CommunicationUserIdentifier)communicationIdentifier).getId());
}
else if (communicationIdentifier instanceof MicrosoftTeamsUserIdentifier) {
    System.out.println("Teams user: " + ((MicrosoftTeamsUserIdentifier)communicationIdentifier).getUserId());
}
else if (communicationIdentifier instanceof PhoneNumberIdentifier) {
    System.out.println("Phone number: " + ((PhoneNumberIdentifier)communicationIdentifier).getPhoneNumber());
}
else if (communicationIdentifier instanceof UnknownIdentifier) {
    System.out.println("Unkown user: " + ((UnknownIdentifier)communicationIdentifier).getId());
}
else {
    // be careful here whether you want to throw because a new SDK version
        // can introduce new identifier types
}
```

## Raw ID representation

Sometimes you need to serialize an identifier to a flat string. For example, if you want to store the identifier in a database table or if you'd like to use it as a URL parameter.

For that purpose, identifiers have another representation called `RawId`. An identifier can always be translated to its corresponding raw ID, and a valid raw ID can always be converted to an identifier.

Since `azure-communication-common 1.2.0` the SDK helps with the conversion:

```java
// get an identifier's raw Id
String rawId = communicationIdentifier.getRawId();

// create an identifier from a given raw Id
CommunicationIdentifier identifier = CommunicationIdentifier.fromRawId(rawId);
```

An invalid raw ID will just convert to an `UnknownIdentifier` in the SDK and any validation only happens service-side.