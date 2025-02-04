.. include:: ../common/common_definitions.rst

.. _wallet-trust-evidence.rst:

Wallet Trust Evidence
+++++++++++++++++++++

In the context of eIDAS 2.0, there is a requirement which is stated "An attestation that is issued by the Wallet Provider is needed to ensure that the keys used for key binding of credentials 
are really reside in a trustworthy Wallet Secure Cryptographic Device (WSCD)." We called this attestation Wallet Trust Evidence (WTE). 


Requirements
------------

The requirements for the Wallet Trust Evidence are defined below:

- The Wallet Trust Evidence MUST contain one or multiple attested credential's public key that are coming from the same WSCD.
- The Wallet Trust Evidence MUST use the signed JSON Web Token (JWT) format;
- The Wallet Trust Evidence MUST provide all the relevant information to attest the trustworthiness of the WSCD component where the private keys are stored.
- The Wallet Trust Evidence MUST be signed by the Wallet Provider or the WSCD component.
- The Wallet Provider MUST ensure the trustworthiness of the WSCD, preventing any attempts at manipulation or falsification by unauthorized third parties. 
- The Wallet Trust Evidence MUST be used once and during its validity period, with the need to request new attestations with each interaction.
- The Wallet Trust Evidence MUST be short-lived and MUST have an expiration date/time, after which it SHOULD no longer be considered valid.
- The Wallet Trust Evidence MUST NOT be issued by the Wallet Provider or the WSCD component if the WSCD trustworthiness is not guaranteed. In this case, the Wallet Instance MUST be revoked.
- The Wallet Provider MUST offer a set of services, exclusively available to its Wallet Solution instances, for the issuance of Wallet Trust Evidence.



Wallet Trust Evidence Issuance
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This section describes the Wallet Trust Evidence format and how the Wallet Provider issues it.

.. figure:: ../../images/wallet_trust_evidence_acquisition.svg
   :name: Sequence Diagram for Wallet Trust Evidence acquisition
   :alt: The figure illustrates the sequence diagram for issuing a Wallet Trust Evidence, with the steps explained below.
   :target: https://www.plantuml.com/plantuml/uml/VLH1Rziy3BthLn3v3bzhU9S1ss70Xgv5rdOTYgBDrak0WB7PM8WZUHATBllr8-KqSOB0XmIMzVZu-FJaYyWOk5tL1blshbtDAPZAbWGltlFS_p7_DmAmXMtGiJ5Oi0ym-XafZ00Zj57mFGICdh6kYU7M2RChAA6mQS08sMxt8VYrD0bmYSMIN3c2_txOHSMNTKjmo3Vn0e2nAnjl7IUwUR4ynDnxwNI8SGeIPj0PZCfyzqLaV897-jrIP40e0fNas68DDiOMbSCt592jTy0LyjG5GTj04RBiLZ2YU19QgHwhV2d8ChN4hf59fpJos_QvggXOWdsHogkmQTWl0ZQLBU06G_cAWU2EDZ7Bf3VW6csDyvfCZ-2Qd6eXsNP0JKKhMTPzrKlQG8CsI8llptT2TLQ4MTFEQrlae8yX2Jit9ZZF17vDGLNcQeuuVdjzCxb-78_lJIVs-714cTeC_aNS8FX6vPivACRwEQDrO3d2YXXBP5J3Kokp7KGRwIGi4fq_yYiTaVxjSTt4u1t1L6PSaGQiLvlGdJ-zjoKTSffJDWfU_9fL62jn2YCytNnz_-4Zd2MM77RMdGzKOumKr06XY7RXIFAr6JnX0KxTKV5CIv7RGBBxEH6TlMdBuJMTWYmwafdkC2xwiav6SPViLyiL3BJCzzRbKpSa7YQuwF0xT_Q3QvjUBXCcKX68isohDPtgazx2GSLcdmcfw8TLbldfhDekaqTV6usiTLPlX_rBPShfMfvBqqRh5Z0mgbYiy27Zzl4MJTlfVYaxSY-a9pS7I4_2nSlkIZ_uXzY3N0KImC3NQ7z11a0b7HZUNqkbkP0vsrNz3m00

**Step 1**: The User initiates a new operation that necessitates the acquisition of a Wallet Attestation.

**Steps 2-3**: The Wallet Instance checks if a Cryptographic Hardware Key exists and generates an ephemeral asymmetric key pair. The Wallet Instance also:

  1. MUST ensure that Cryptographic Hardware Keys exist. If they do not exist, it is necessary to reinitialize the Wallet.
  2. MUST generates an ephemeral asymmetric key pair whose public key will be linked with the Wallet Attestation.
  3. MUST check if Wallet Provider is part of the federation and obtain its metadata.


**Steps 4-6**: The Wallet Instance solicits a one-time "challenge" from the Wallet Provider Backend. This "challenge" takes the form of a "nonce," which is required to be unpredictable and serves as the main defense against replay attacks. The backend MUST produce the "nonce" in a manner that ensures its single-use within a predetermined time frame.

.. code-block:: http

    GET /nonce HTTP/1.1
    Host: walletprovider.example.com

.. code-block:: http

    HTTP/1.1 200 OK
    Content-Type: application/json

    {
      "nonce": "d2JhY2NhbG91cmVqdWFuZGFt"
    }

**Step 7**: The Wallet Instance performs the following actions:

  * Creates a ``client_data``, a JSON structure that includes the challenge and the thumbprint of ephemeral public ``jwk``.
  * Computes a ``client_data_hash`` by applying the ``SHA256`` algorithm to the ``client_data``.

Below a non-normative example of the ``client_data``.

.. code-block:: json

  {
    "challenge": "0fe3cbe0-646d-44b5-8808-917dd5391bd9",
    "jwk_thumbprint": "vbeXJksM45xphtANnCiG6mCyuU4jfGNzopGuKvogg9c"
  }

**Steps 8-10**: The Wallet Instance takes the following steps:

  *  It produces an hardware_signature by signing the ``client_data_hash`` with the Wallet Hardware's private key, serving as a proof of possession for the Cryptographic Hardware Keys.
  *  It requests the Device Integrity Service to create an ``integrity_assertion`` linked to the ``client_data_hash``.
  *  It receives a signed ``integrity_assertion`` from the Device Integrity Service, authenticated by the OEM.

.. note:: ``integrity_assertion`` is a custom payload generated by Device Integrity Service, signed by device OEM and encoded in base64 to have uniformity between different devices.

**Steps 11-12**: The Wallet Instance:

  * Constructs the Wallet Attestation Request in the form of a JWT. This JWT includes the ``integrity_assertion``, ``hardware_signature``, ``challenge``, ``hardware_key_tag``, ``cnf`` and other configuration related parameters (see :ref:`Table of the Wallet Attestation Request Body <table_wallet_attestation_request_claim>` below) and is signed using the private key of the initially generated ephemeral key pair.
  * Submits the Wallet Attestation Request to the token endpoint of the Wallet Provider Backend.

Below an non-normative example of the Wallet Attestation Request JWT without encoding and signature applied:

.. code-block::

  {
    "alg": "ES256",
    "kid": "vbeXJksM45xphtANnCiG6mCyuU4jfGNzopGuKvogg9c",
    "typ": "war+jwt"
  }
  .
  {
    "iss": "https://wallet-provider.example.org/instance/vbeXJksM45xphtANnCiG6mCyuU4jfGNzopGuKvogg9c",
    "sub": "https://wallet-provider.example.org/",
    "challenge": "6ec69324-60a8-4e5b-a697-a766d85790ea",
    "hardware_signature": "KoZIhvcNAQcCoIAwgAIB...redacted",
    "integrity_assertion": "o2NmbXRvYXBwbGUtYXBwYX...redacted",
    "hardware_key_tag": "WQhyDymFKsP95iFqpzdEDWW4l7aVna2Fn4JCeWHYtbU=",
    "cnf": {
      "jwk": {
        "crv": "P-256",
        "kty": "EC",
        "x": "4HNptI-xr2pjyRJKGMnz4WmdnQD_uJSq4R95Nj98b44",
        "y": "LIZnSB39vFJhYgS3k7jXE4r3-CoGFQwZtPBIRqpNlrg"
      }
    },
    "vp_formats_supported": {
        "jwt_vc_json": {
          "alg_values_supported": ["ES256K", "ES384"]
        },
        "jwt_vp_json": {
          "alg_values_supported": ["ES256K", "EdDSA"]
        },
      },
    },
    authorization_endpoint": "https://wallet-solution.digital-strategy.europa.eu/authorization",
    "response_types_supported": [
      "vp_token"
    ],
    "response_modes_supported": [
      "form_post.jwt"
    ],
    "request_object_signing_alg_values_supported": [
      "ES256"
    ],
    "presentation_definition_uri_supported": false,
    "iat": 1686645115,
    "exp": 1686652315
  }

The Wallet Instance MUST do an HTTP request to the Wallet Provider's `token endpoint`_,
using the method `POST <https://datatracker.ietf.org/doc/html/rfc6749#section-3.2>`__.

The **token** endpoint (as defined in `RFC 7523 section 4`_) requires the following parameters
encoded in ``application/x-www-form-urlencoded`` format:

* ``grant_type`` set to ``urn:ietf:params:oauth:grant-type:jwt-bearer``;
* ``assertion`` containing the signed JWT of the Wallet Attestation Request.

.. code-block:: http

    POST /token HTTP/1.1
    Host: wallet-provider.example.org
    Content-Type: application/x-www-form-urlencoded

    grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
    &assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6ImtoakZWTE9nRjNHeG...

**Steps 13-17**: The Wallet Provider Backend assesses the Wallet Attestation Request and issues a Wallet Attestation, if the requirements described below are satisfied:

    1. It MUST check the Wallet Attestation Request contains all the defined HTTP Request header parameters according to :ref:`Table of the Wallet Attestation Request Header <table_wallet_attestation_request_claim>`.
    2. It MUST verify that the signature of the received Wallet Attestation Request is valid and associated with public ``jwk``.
    3. It MUST verify that the ``challenge`` was generated by  Wallet Provider and has not already been used.
    4. It MUST check that there is a Wallet Instance registered with that ``hardware_key_tag`` and that it is still valid.
    5. It MUST reconstruct the ``client_data`` via the ``challenge`` and the ``jwk`` public key, to validate ``hardware_signature`` via the Cryptographic Hardware Key public key registered and associated with the Wallet Instance.
    6. It MUST validate the ``integrity_assertion`` as defined by the device manufacturers' guidelines. The list of checks that the Wallet Provider MUST perform are defined by the operating system manufacturers documentation.
    7. It MUST verify that the device in use has no security flaws and reflects the minimum security requirements defined by the Wallet Provider.
    8. It MUST check that the URL in ``iss`` parameter is equal to the URL identifier of Wallet Provider.

If all checks are passed, Wallet Provider issues a Wallet Attestation with an expiration limited to 24 hours.

Below an non-normative example of the Wallet Attestation without encoding and signature applied:

.. code-block::

    {
    "alg": "ES256",
    "kid": "5t5YYpBhN-EgIEEI5iUzr6r0MR02LnVQ0OmekmNKcjY",
    "trust_chain": [
      "eyJhbGciOiJFUz...6S0A",
      "eyJhbGciOiJFUz...jJLA",
      "eyJhbGciOiJFUz...H9gw",
    ],
    "typ": "wallet-attestation+jwt",
  }
  .
  {
    "iss": "https://wallet-provider.example.org",
    "sub": "vbeXJksM45xphtANnCiG6mCyuU4jfGNzopGuKvogg9c",
    "aal": "https://trust-list.eu/aal/high",
    "cnf":
    {
      "jwk":
      {
        "crv": "P-256",
        "kty": "EC",
        "x": "4HNptI-xr2pjyRJKGMnz4WmdnQD_uJSq4R95Nj98b44",
        "y": "LIZnSB39vFJhYgS3k7jXE4r3-CoGFQwZtPBIRqpNlrg"
      }
    },
    "authorization_endpoint": "https://wallet-solution.digital-strategy.europa.eu/authorization",
    "response_types_supported": [
      "vp_token"
    ],
    "response_modes_supported": [
      "form_post.jwt"
    ],
    "vp_formats_supported": {
        "dc+sd-jwt": {
            "sd-jwt_alg_values": [
                "ES256",
                "ES384"
            ]
        }
    },
    "request_object_signing_alg_values_supported": [
      "ES256"
    ],
    "presentation_definition_uri_supported": false,
    "iat": 1687281195,
    "exp": 1687288395
  }

**Step 18**: The response is returned by the Wallet Provider. If successful, the HTTP response code MUST be set with the value ``200 OK`` and contain the Wallet Attestation signed by the Wallet Provider. The Wallet Instance therefore performs security, integrity and trust verification about the Wallet Attestation and its issuer.


Below is a non-normative example of the response.

.. code-block:: http

    HTTP/1.1 200 OK
    Content-Type: application/jwt

    eyJhbGciOiJFUzI1NiIsInR5cCI6IndhbGx ...


.. _table_wallet_attestation_request_claim:

Wallet Attestation Request
~~~~~~~~~~~~~~~~~~~~~~~~~~

The JOSE header of the Wallet Attestation Request JWT MUST contain:

.. list-table::
    :widths: 20 60 20
    :header-rows: 1

    * - **JOSE header**
      - **Description**
      - **Reference**
    * - **alg**
      - A digital signature algorithm identifier such as per IANA "JSON Web Signature and Encryption Algorithms" registry. It MUST be one of the supported algorithms listed in the Section `Cryptographic Algorithms <algorithms.html>`_ and MUST NOT be set to ``none`` or any symmetric algorithm (MAC) identifier.
      - :rfc:`7516#section-4.1.1`.
    * - **kid**
      -  Unique identifier of the ``jwk`` used by the Wallet Provider to sign the Wallet Attestation, essential for matching the Wallet Provider's cryptographic public key needed for signature verification.
      - :rfc:`7638#section_3`.
    * - **typ**
      -  It MUST be set to ``var+jwt``
      -

The body of the Wallet Attestation Request JWT MUST contain:

.. list-table::
    :widths: 20 60 20
    :header-rows: 1

    * - **Claim**
      - **Description**
      - **Reference**
    * - **iss**
      - Identifier of the Wallet Provider concatenated with thumbprint of the JWK in the ``cnf`` parameter.
      - :rfc:`9126` and :rfc:`7519`.
    * - **aud**
      - It MUST be set to the identifier of the Wallet Provider.
      - :rfc:`9126` and :rfc:`7519`.
    * - **exp**
      - UNIX Timestamp with the expiry time of the JWT.
      - :rfc:`9126` and :rfc:`7519`.
    * - **iat**
      - REQUIRED. UNIX Timestamp with the time of JWT issuance.
      - :rfc:`9126` and :rfc:`7519`.
    * - **challenge**
      - Challenge data obtained from ``nonce`` endpoint
      -
    * - **hardware_signature**
      - The signature of ``client_data`` obtained using Cryptographic Hardware Key base64 encoded.
      -
    * - **integrity_assertion**
      - The integrity assertion obtained from the **Device Integrity Service** with the holder binding of ``client_data``.
      -
    * - **hardware_key_tag**
      - Unique identifier of the **Cryptographic Hardware Keys**
      -
    * - **cnf**
      - JSON object, containing the public part of an asymmetric key pair owned by the Wallet Instance.
      - :rfc:`7800`
    * - **vp_formats_supported**
      - JSON object with name/value pairs, identifying a Credential format supported by the Wallet.
      -
    * - **authorization_endpoint**
      - URL of the Wallet Authorization Endpoint, it can be a universal link or a custom url-scheme.
      -
    * - **response_types_supported**
      - JSON array containing a list of the OAuth 2.0 ``response_type`` values.
      -
    * - **response_modes_supported**
      - JSON array containing a list of the OAuth 2.0 "response_mode" values that this authorization server supports.
      - :rfc:`8414`
    * - **request_object_signing_alg_values_supported**
      - JSON array containing a list of the signing algorithms (alg values) supported.
      -
    * - **presentation_definition_uri_supported**
      - Boolean value specifying whether the Wallet Instance supports the transfer of presentation_definition by reference. MUST be set to false.
      -

.. _table_wallet_attestation_claim:

Wallet Attestation
~~~~~~~~~~~~~~~~~~

The JOSE header of the Wallet Attestation JWT MUST contain:

.. list-table::
    :widths: 20 60 20
    :header-rows: 1

    * - **JOSE header**
      - **Description**
      - **Reference**
    * - **alg**
      - A digital signature algorithm identifier such as per IANA "JSON Web Signature and Encryption Algorithms" registry. It MUST be one of the supported algorithms listed in the Section `Cryptographic Algorithms <algorithms.html>`_ and MUST NOT be set to ``none`` or any symmetric algorithm (MAC) identifier.
      - :rfc:`7516#section-4.1.1`.
    * - **kid**
      -  Unique identifier of the ``jwk`` inside the ``cnf`` claim of Wallet Instance as base64url-encoded JWK Thumbprint value.
      - :rfc:`7638#section_3`.
    * - **typ**
      -  It MUST be set to ``wallet-attestation+jwt``
      -  `OPENID4VC-HAIP`_
    * - **trust_chain**
      - Sequence of Entity Statements that composes the Trust Chain related to the Relying Party.
      - `OID-FED`_ Section 4.3 *Trust Chain Header Parameter*.

The body of the Wallet Attestation JWT MUST contain:

.. list-table::
    :widths: 20 60 20
    :header-rows: 1

    * - **Claim**
      - **Description**
      - **Reference**
    * - **iss**
      - Identifier of the Wallet Provider
      - :rfc:`9126` and :rfc:`7519`.
    * - **sub**
      - Identifier of the Wallet Instance which is the thumbprint of the Wallet Instance JWK contained in the ``cnf`` claim.
      - :rfc:`9126` and :rfc:`7519`.
    * - **exp**
      - UNIX Timestamp with the expiry time of the JWT.
      - :rfc:`9126` and :rfc:`7519`.
    * - **iat**
      - UNIX Timestamp with the time of JWT issuance.
      - :rfc:`9126` and :rfc:`7519`.
    * - **cnf**
      - JSON object, containing the public part of an asymmetric key pair owned by the Wallet Instance.
      - :rfc:`7800`
    * - **aal**
      - JSON String asserting the authentication level of the Wallet and the key as asserted in the cnf claim.
      -
    * - **authorization_endpoint**
      - URL of the Wallet Authorization Endpoint, it can be a universal link or a custom url-scheme.
      -
    * - **response_types_supported**
      - JSON array containing a list of the OAuth 2.0 ``response_type`` values.
      -
    * - **response_modes_supported**
      - JSON array containing a list of the OAuth 2.0 "response_mode" values that this authorization server supports.
      - :rfc:`8414`
    * - **vp_formats_supported**
      - JSON object with name/value pairs, identifying a Credential format supported by the Wallet.
      -
    * - **request_object_signing_alg_values_supported**
      - JSON array containing a list of the signing algorithms (alg values) supported.
      -
    * - **presentation_definition_uri_supported**
      - Boolean value specifying whether the Wallet Instance supports the transfer of presentation_definition by reference. MUST be set to false.
      -
    * - **client_id_schemes_supported**
      - Array of JSON Strings containing the values of the Client Identifier schemes that the Wallet supports.
      - `OpenID4VP`_

Revocations
~~~~~~~~~~~~~~~~~~
As mentioned in the *Wallet Instance initialization and registration* section above, a Wallet Instance is bound to a Wallet Hardware Key and it's uniquely identified by it.
The Wallet Instance SHOULD send its public Wallet Hardware Key with the Wallet Provider, thus the Wallet Provider MUST identify a Wallet Instance by its Wallet Hardware Key.

When a Wallet Instance is not usable anymore, the Wallet Provider MUST revoke it. The revocation process is a unilateral action taken by the Wallet Provider, and it MUST be performed when the Wallet Instance is in the `Operational` or `Valid` state.
A Wallet Instance becomes unusable for several reasons, such as: the User requests the revocation, the Wallet Provider detects a security issue, or the Wallet Instance is no longer compliant with the Wallet Provider's security requirements.

The details of the revocation mechanism used by the Wallet Provider as well as the data model for maintaining the Wallet Instance references is delegated to the Wallet Provider's implementation.

According to ARF, `Section 6.5.4 <https://github.com/eu-digital-identity-wallet/eudi-doc-architecture-and-reference-framework/blob/main/docs/arf.md#654-wallet-instance-management>`_ and more specifically in `Topic 38 <https://github.com/eu-digital-identity-wallet/eudi-doc-architecture-and-reference-framework/blob/main/docs/annexes/annex-2/annex-2-high-level-requirements.md#a2338-topic-38---wallet-instance-revocation>`_ the Wallet Instance can be revoked by the following entities:

  1. Its owner, the User
  2. Wallet Provider
  3. PID Provider

During the *Wallet Instance initialization and registration* phase the Wallet Provider MAY associate the Wallet Instance with a specific User, subject to obtaining the User's consent. The Wallet Provider MUST evaluate the operating system and general technical capabilities of the device to check compliance with the technical and security requirements and to produce the Wallet Instance metadata.
When the User consents to being linked with the Wallet Instance, they gain the ability to directly request Wallet revocation from the Wallet Provider, and it also allows the Wallet Provider to revoke the Wallet Instance associated with that User.

Regarding the reasons for revoking a Wallet Instance, the following scenarios may occur:

- The smartphone is lost;
- The smartphone has been compromised (e.g., a malicious actor gains control of the smartphone);
- The smartphone has been reset to factory settings;
- Any other scenarios where the User loses the control of the Wallet Instance.

If any of the previous scenarios occur, the Wallet Instance **MUST** be revoked.
To allow the User to revoke the Wallet Instance, the Wallet Provider (WP) **MUST**  offer a remote service, such as a web page, where the User can authenticate and request the revocation of a previously activated Wallet Instance.

.. _token endpoint: wallet-solution.html#wallet-attestation
.. _Wallet Attestation Request: wallet-attestation.html#format-of-the-wallet-attestation-request
.. _Wallet Attestation: wallet-attestation.html#format-of-the-wallet-attestation
.. _RFC 7523 section 4: https://www.rfc-editor.org/rfc/rfc7523.html#section-4
.. _RFC 8414 section 2: https://www.rfc-editor.org/rfc/rfc8414.html#section-2
.. _Wallet Provider metadata: wallet-solution.html#wallet-provider-metadata
.. _Play Integrity API: https://developer.android.com/google/play/integrity?hl=it
.. _DeviceCheck: https://developer.apple.com/documentation/devicecheck
.. _OAuth 2.0 Nonce Endpoint: https://datatracker.ietf.org/doc/draft-demarco-oauth-nonce-endpoint/
.. _ARF: https://github.com/eu-digital-identity-wallet/eudi-doc-architecture-and-reference-framework
