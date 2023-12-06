---
author: "Greg Simons"
title: "Spring Security private key jwt with AWS KMS"
description: "How to configure spring security to use private key jwt with AWS KMS"
draft: false
date: 2023-12-05
tags: ["spring", "AWS"]
categories: ["spring", "AWS"]
ShowToc: true
TocOpen: true
---

As part of our proof of concepts in to the adoption of One Login we setup a Spring Security OAuth 2.0 
demo that tested out the [integration guide](https://docs.sign-in.service.gov.uk/integrate-with-integration-environment/integrate-with-code-flow/#make-a-token-request)
provided by Government Digital Service (GDS).

Spring security has long had great OAuth2.0 support from both the server and client elements. Over the last year spring security 
added support for the private_key_jwt client authentication method as part of the authorization code grant flow.
[Spring Security GitHub ref](https://github.com/spring-projects/spring-security/pull/9933)

The configuration for the key signing requires both the public and private key to be available to the application. 
Key Management Service (KMS) by Amazon Web Services (AWS) provides services to centrally manage your keys for encryption 
and signing and is an option to allow a more centralised mechanism for key rotation and policy management.

In order to add this support in to Spring it requires a number of customisations to be made which will be explained in 
this post. If your AuthZ server also supports elliptic-curve digital signature algorithms (ECDSA) for the ID token I will 
outline the additional Bean configuration required as Spring security defaults to RS256 supported with RSA keys.

## private_key_jwt
The private key jwt client authentication method requires a jwt token to be sent alongside client id, code as a client 
assertion to the token endpoint. This jwt will need to be signed and then sent in a client_assertion parameter to the 
auth server. A number of algorithms are supported by various auth servers and can be found by visiting the configuration 
endpoint of the auth server.

Full details of the spec can be found here [rfc 7523](https://www.rfc-editor.org/rfc/rfc7523.html)

The first configuration change required is to switch out the default access token response client as part of the 
HttpSecurity config. This provides the ability to customise the token endpoint request parameters to enrich with the client 
assertion signed by kms.

```java
@Bean
public Security FilterChain filterchain(HttpSecurity http) throws Exception {
  ....
  http.oauth2Login(oauth2Login ->
    oauthLogin.tokenEndpoint(tokenEndpoint ->
      tokenEndpoint.accessTokenResponseClient(accessTokenResponseClient))).
    .oauth2Client.authorizationCodeGrant(authorizationCodeGrant ->
    authorizationCodeGrant.accessTokenResponseClient(accessTokenResponseClient)));

  ....
  http.build();
}
```

## OAuth2AccessTokenResponseClient
In order to override the request entity handling to add support for the kms signing you need to add a custom request 
entity converter to the access token response client that you create.

```java
...
@Bean
public OAuth2AccessTokenResponseClient<OAuthAuthorizationCodeGrantRequest accessTokenResponseClient() {
  ....
  DefaultAuthorizationCodeTokenResponseClient client = new DefaultAuthorizationCodeTokenResponseClient();
  client.setRequestEntityConverter(converter);
  return client;
}
```
From this point it is now necessary to create the converter:

    It is important to note whether or not you are creating the beans or are they being managed and created by Spring 
    as you could incur a number of errors if the objects are manually instantiated.

The converter is the main class that allows the overriding of the request parameters.

```java
@Component
public class CustomKMSJWTClientAuthNConverter implements Converter<OAuth2AuthorizationCodeGrantRequest, RequestEntity<?>> {

  private OAuth2AuthorizationCodeGrantRequestEntityConverter defaultConverter;

  public CustomKMSJWTClientAuthNConverter () {
    defaultConverter = new OAuth2AuthorizationCodeGrantRequestEntityConverter();
  }

  public RequestEntity<?> convert(OAuth2AuthorizationCodeGrantRequest req) {
    String signedJWT = createSignedJwt();
    RequestEntity<?> entity = defaultConverter.convert(req);
    MultiValueMap<String, String> parameters = (MultiValueMap<String, String>) entity.getBody();
    parameters.set("client_assertion_type", "urn:ietf:params:oauth:client-assertion-type:jwt-bearer");
    parameters.set("client_assertion", signedJWT);
    return new RequestEntity<>(parameters, entity.getHeaders(), entity.getMethod(), entity.getUrl());
  }
}
```

## Creating a signed JWT
The next step is to create the JWT and use the KMS API to sign the token.

You will need to setup the AWS SDK and KMS Client. For example, I exported the AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN 
and AWS_ACCESS_KEY_ID environment variables and set them on the command line. There are a number of alternatives supported 
in spring that can be found here: [AWS SDK on Spring](https://aws.amazon.com/blogs/opensource/getting-started-with-spring-boot-on-aws-part-1/)

```java
kmsClient = KmsClient.builder().region(Region.of("....."))
      .credentialsProvider(DefaultCredentialsProvider.create()
      .build();
```

```java
// tbc values can be configured with your auth server

  public String createSignedJwt() {
    JWTClaimSet.Builder claimSetBuilder = new JWTClaimSet.Builder().audience('tbc').issuer('tbc').subject('tbc').expirationTime(tbc).issueTime(Date.from(Instant.now())).jwtId('tbc');

    JWTClaimSet claimSet = claimSetBuilder.build();
    return sign(claimSet);
  }
```

We now have a JWT token ready to sign and verify with KMS.

```java
public String sign(JWTClaimSet claimSet) {
      ....
      // choose a token alg based on what is supported by your auth Server
      JWSHeader header = new JWSHeader(TOKEN_ALG);

      Base64URL encodedHeader = header.toBase64URL();
      Base64URL encodedClaims = Base64URL.encode(claimSet.toString());
      // create String to sign with KMS
      String signingString = encodedHeader + "." + encodedClaims;
      ....
    }
  }
```

We now need to use the AWS SDK to create the signing request to pass to KMS to get the signed JWT.

Key Id is the id of your key from AWS.
Signing alg can be selected from a predefined set using
software.amazon.awssdk.services.kms.model.SigningAlgorithmSpec;

```java
 ....
  SignRequest signRequest = SignRequest.builder()

  .message(SDKBytes.fromByteArray(signingString.getBytes()))
  .keyId(KEY_ID)
  .signingAlgorithm(SIGNING_ALG)
  .build();

  SignResponse response = kmsClient.sign(signRequest);
  String signature = Base64.encode(response.signature().asByteArray()).toString();
  return signature;
```

The above call now provides a Base64 encoded version of the signed string that is attached to the request parameters for 
the invocation of the token endpoint.

## ID Token Verification
Depending on the supported algorithms for the id token you might find that the ID Token is signed with a different 
algorithm than the default RS256 that spring supports. In order to override this setting you can again create a custom 
Bean that sets alternative signatures using the JWSAlgorithmResolver.

```java
@Bean
public JwtDecoderFactory<ClientRegistration> idTokenDecoderFactory() {
  OidcTokenDecoderFactory idTokenDecoderFactory = new OidcTokenDecoderFactory();
  idTokenDecoderFactory.setJwsAlgorithmResolver(clientRegistration -> SignatureAlgorithm.RS512);
return idTokenDecoderFactory;
}
```