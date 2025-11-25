# Digikoppeling WUS: The Hybrid SOAP Challenge in Java/Kotlin

## Understanding the SOAP 1.1 + SOAP 1.2 Mix and Solutions

## Executive Summary

Digikoppeling is the Dutch government's standardized infrastructure for secure data exchange between public sector organizations. The WUS (Web Services) profile presents a unique technical challenge by **mixing elements from SOAP 1.1 and SOAP 1.2 in a single message structure**.

Specifically, WUS mandates:
- **SOAP 1.1 envelope** (from WS-I Basic Profile 1.2)
- **SOAP 1.2 features embedded within**: WS-Addressing and WS-Security 1.1 (from WS-I BSP 1.1)
- **Optional MTOM attachments** for binary data
- **WS-Addressing callback support** for asynchronous communication patterns

This "hybrid SOAP" approach violates the strict version segregation built into standard Java web service frameworks (Apache Axis, CXF, Metro, Spring-WS, JBossWS), which expect all message components to use a consistent SOAP version.

This document explores:
- **The technical nature of the hybrid SOAP problem**: Why WUS mixes SOAP versions
- **Why standard frameworks cannot handle it**: Version validation and namespace coherence assumptions
- **A practical solution using JAR shading**: Creating a customized Apache CXF that accepts the hybrid format
- **Implementation guidance for Java and Kotlin**: Code examples, configuration, and best practices
- **Integration with middleware platforms**: ESB, Spring Integration, Apache Camel patterns
- **MTOM and asynchronous callback patterns**: Optional features for attachments and long-running operations

The recommended approach is to create a **shaded, customized version of Apache CXF** that relaxes version validation for WUS-specific messages, while maintaining a standard CXF installation for other services. This enables organizations to meet Digikoppeling compliance requirements without compromising their broader service architecture.

## Table of Contents

1. [Understanding Digikoppeling](#understanding-digikoppeling)
2. [The WUS Profile](#the-wus-profile)
3. [The SOAP Version Mixing Problem](#the-soap-version-mixing-problem)
4. [Affected Java Frameworks](#affected-java-frameworks)
5. [The Shading Solution](#the-shading-solution)
6. [Integration with Middleware](#integration-with-middleware)
7. [Java and Kotlin Considerations](#java-and-kotlin-considerations)
8. [Practical Implementation](#practical-implementation)
9. [MTOM Attachments and Asynchronous Callbacks](#mtom-attachments-and-asynchronous-callbacks)
10. [Recommendations](#recommendations)

## Understanding Digikoppeling

Digikoppeling is a connection standard mandated for Dutch public sector organizations to facilitate secure and reliable data exchange. It provides:

- **Standardized protocols** for data exchange
- **Security requirements** based on WS-Security
- **Multiple profiles** for different use cases
- **Interoperability** between government systems

The standard ensures that organizations can communicate regardless of their underlying technology stack, promoting vendor neutrality and long-term sustainability.

## The WUS Profile

The WUS (Web Services) profile is one of Digikoppeling's key profiles, designed for synchronous request-response patterns. However, it presents a unique technical challenge: **it mixes elements from both SOAP 1.1 and SOAP 1.2 in a single standard**.

### Core Components and the SOAP Version Mix

The WUS profile is based on:

1. **WS-I Basic Profile 1.2**: Which mandates SOAP 1.1 as the base envelope
2. **WS-I Basic Security Profile 1.1**: Which uses BOTH WS-Security 1.0 and WS-Security 1.1 namespaces
3. **WS-Addressing**: A SOAP 1.2 specification
4. **WS-Security 1.1**: Which references SOAP 1.2 elements and patterns

### The SOAP Version Mixing Problem

Here's the critical issue: **WUS requires using SOAP 1.1 envelopes with SOAP 1.2 features embedded within them**.

Specifically, a WUS message must contain:

- **SOAP 1.1 Envelope** (`http://schemas.xmlsoap.org/soap/envelope/`)
  - As mandated by WS-I Basic Profile 1.2
  - Document/literal binding style
  - SOAP 1.1 fault structure

- **SOAP 1.2 Features** mixed into the SOAP 1.1 envelope:
  - **WS-Addressing headers** (originally designed for SOAP 1.2)
    - `wsa:To`, `wsa:Action`, `wsa:MessageID`, `wsa:RelatesTo`, `wsa:ReplyTo`, `wsa:From`
  - **WS-Security 1.1 elements** (which reference SOAP 1.2 constructs)
    - Security headers with both WS-Security 1.0 and 1.1 namespaces
    - Timestamp, Signature, Encryption elements
  - **BSP 1.1 compliance** (which assumes SOAP 1.2 security model)

This creates what can be called a **"hybrid SOAP message"** - neither purely SOAP 1.1 nor SOAP 1.2, but a specific combination of both.

### Understanding WS-I BSP 1.1's Role in the Mix

According to the Digikoppeling specification, WS-I Basic Security Profile 1.1 explicitly states:

> "Basic Security Profile 1.1 is sinds 2010 november final geworden. Hierin worden zowel de WS-Security 1.0 als de WS-Security 1.1 namespaces beide gebruikt."

Translation: *"Basic Security Profile 1.1 became final in November 2010. It uses BOTH WS-Security 1.0 and WS-Security 1.1 namespaces."*

This is critical because:
- **WS-Security 1.0** was designed for SOAP 1.1
- **WS-Security 1.1** was designed for SOAP 1.2 and introduces SOAP 1.2-style constructs
- **BSP 1.1 uses both** in the same message

### Concrete Example: WUS Message Headers

Here's what a typical WUS request header looks like:

```xml
<soap:Header xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
    
    <!-- WS-Addressing: SOAP 1.2 specification -->
    <wsa:To xmlns:wsa="http://www.w3.org/2005/08/addressing">
        http://service.overheid.nl/wus?OIN=00000001234567890000
    </wsa:To>
    <wsa:Action xmlns:wsa="http://www.w3.org/2005/08/addressing">
        http://example.nl/service/v1/RequestData
    </wsa:Action>
    <wsa:MessageID xmlns:wsa="http://www.w3.org/2005/08/addressing">
        uuid:a1b2c3d4-e5f6-4a5b-8c9d-0e1f2a3b4c5d
    </wsa:MessageID>
    <wsa:ReplyTo xmlns:wsa="http://www.w3.org/2005/08/addressing">
        <wsa:Address>http://www.w3.org/2005/08/addressing/anonymous</wsa:Address>
    </wsa:ReplyTo>
    
    <!-- WS-Security: Mix of 1.0 and 1.1 namespaces -->
    <wsse:Security 
        xmlns:wsse="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd"
        xmlns:wsu="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd"
        soap:mustUnderstand="1">
        
        <!-- Timestamp with UTC -->
        <wsu:Timestamp wsu:Id="TS-1">
            <wsu:Created>2025-11-25T10:30:00.000Z</wsu:Created>
            <wsu:Expires>2025-11-25T10:35:00.000Z</wsu:Expires>
        </wsu:Timestamp>
        
        <!-- Binary Security Token (PKIoverheid certificate) -->
        <wsse:BinarySecurityToken 
            EncodingType="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-soap-message-security-1.0#Base64Binary"
            ValueType="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-x509-token-profile-1.0#X509v3"
            wsu:Id="X509-TOKEN">
            MIIEpDCCA4ygAwIBAgIQYU... (base64 encoded certificate)
        </wsse:BinarySecurityToken>
        
        <!-- Signature (signs Body, WS-Addressing headers, and Timestamp) -->
        <ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#" Id="SIG-1">
            <ds:SignedInfo>
                <ds:CanonicalizationMethod Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
                <ds:SignatureMethod Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256"/>
                
                <!-- Reference to SOAP Body -->
                <ds:Reference URI="#BODY-1">
                    <ds:Transforms>
                        <ds:Transform Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
                    </ds:Transforms>
                    <ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>
                    <ds:DigestValue>abc123...</ds:DigestValue>
                </ds:Reference>
                
                <!-- Reference to WS-Addressing headers -->
                <ds:Reference URI="#wsa-to">
                    <ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>
                    <ds:DigestValue>def456...</ds:DigestValue>
                </ds:Reference>
                
                <!-- Reference to Timestamp -->
                <ds:Reference URI="#TS-1">
                    <ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256"/>
                    <ds:DigestValue>ghi789...</ds:DigestValue>
                </ds:Reference>
            </ds:SignedInfo>
            <ds:SignatureValue>xyz890...</ds:SignatureValue>
            <ds:KeyInfo>
                <wsse:SecurityTokenReference>
                    <wsse:Reference URI="#X509-TOKEN" 
                        ValueType="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-x509-token-profile-1.0#X509v3"/>
                </wsse:SecurityTokenReference>
            </ds:KeyInfo>
        </ds:Signature>
        
        <!-- SignatureConfirmation (WS-Security 1.1 feature for SOAP 1.2) -->
        <wsse11:SignatureConfirmation 
            xmlns:wsse11="http://docs.oasis-open.org/wss/oasis-wss-wssecurity-secext-1.1.xsd"
            wsu:Id="SC-1" 
            Value="abc123..."/>
    </wsse:Security>
</soap:Header>
```

Notice how this single header contains:
1. **SOAP 1.1 envelope namespace**: `http://schemas.xmlsoap.org/soap/envelope/`
2. **WS-Addressing (SOAP 1.2)**: `http://www.w3.org/2005/08/addressing`
3. **WS-Security 1.0 namespaces**: `oasis-200401-wss-*`
4. **WS-Security 1.1 namespace**: `oasis-wss-wssecurity-secext-1.1.xsd`

### Required WUS Components

According to the Digikoppeling specification (Voorschrift WA001), these WS-Addressing fields are **mandatory**:

| Field | Usage | Namespace |
|-------|-------|-----------|
| `wsa:To` | Endpoint address (may include OIN parameter) | SOAP 1.2 (WS-Addressing) |
| `wsa:Action` | Operation to invoke (from WSDL) | SOAP 1.2 (WS-Addressing) |
| `wsa:MessageID` | Unique message identifier (UUID) | SOAP 1.2 (WS-Addressing) |
| `wsa:RelatesTo` | In response, refers to request MessageID | SOAP 1.2 (WS-Addressing) |
| `wsa:ReplyTo` | Reply endpoint (default: anonymous) | SOAP 1.2 (WS-Addressing) |
| `wsa:From` | Sender address (optional, may include OIN) | SOAP 1.2 (WS-Addressing) |

All of these use the SOAP 1.2 WS-Addressing namespace, yet they must appear in a SOAP 1.1 envelope.

### The OIN (Organisatie Identificatie Nummer) Pattern

WUS adds another Dutch-specific requirement: the OIN (Organization Identification Number) can be included as a query parameter:

```xml
<wsa:To>http://service.overheid.nl/wus?OIN=00000001234567890000</wsa:To>
<wsa:From>
    <wsa:Address>http://client.overheid.nl?OIN=00000009876543210000</wsa:Address>
</wsa:From>
```

This allows services to identify the sending and receiving organizations at the message level, crucial for Dutch government interoperability.

### WS-Addressing Callback Pattern (Asynchronous Communication)

While the WUS profiles are described as "synchronous exchanges" (Best Effort), the specification actually **supports asynchronous callback patterns** through WS-Addressing.

**Synchronous vs Asynchronous ReplyTo:**

According to the specification (Voorschrift WA001):

> "Voor synchrone communicatie t.b.v. bevragingen zal het replyTo veld gevuld zijn met de waarde `http://www.w3.org/2005/08/addressing/anonymous` of het element volledig weglaten."

Translation: *"For synchronous communication for queries, the replyTo field will be filled with the value `http://www.w3.org/2005/08/addressing/anonymous` or the element can be completely omitted."*

However, the specification also allows:

> "De waarde van het adres element kan hetzij een absolute URI zijn of `http://www.w3.org/2005/08/addressing/anonymous`."

Translation: *"The value of the address element can be either an absolute URI or `http://www.w3.org/2005/08/addressing/anonymous`."*

This means **asynchronous callbacks are possible** by specifying an actual endpoint URI in `wsa:ReplyTo`.

**Synchronous Pattern (Default):**

```xml
<wsa:ReplyTo>
    <wsa:Address>http://www.w3.org/2005/08/addressing/anonymous</wsa:Address>
</wsa:ReplyTo>
```
Or simply omit the `wsa:ReplyTo` element entirely (defaults to anonymous).

**Asynchronous Callback Pattern:**

```xml
<wsa:ReplyTo>
    <wsa:Address>https://client.overheid.nl/callback-endpoint?OIN=00000009876543210000</wsa:Address>
</wsa:ReplyTo>
```

**Real-World Example:**

The Kadaster KLIC B2B-koppeling uses this pattern:

> "Het antwoordadres is een webservice endpoint dat meegegeven moet worden via het wsa:ReplyTo attribuut in de WS-Addressing header."

Translation: *"The reply address is a webservice endpoint that must be provided via the wsa:ReplyTo attribute in the WS-Addressing header."*

**Implications for Implementation:**

| Aspect | Synchronous | Asynchronous Callback |
|--------|-------------|----------------------|
| **ReplyTo** | `anonymous` or omitted | Actual callback endpoint URL |
| **Response** | Immediate HTTP response | Separate HTTP request to callback endpoint |
| **Connection** | Single HTTP connection | Two separate connections |
| **Use Cases** | Queries, fast operations | Long-running processes, heavy processing |
| **Complexity** | Lower | Higher (requires hosting callback endpoint) |

**Asynchronous Callback Flow:**

```
Client                          Service Provider
  |                                    |
  |--- Request (ReplyTo=callback) --->|
  |                                    |
  |<--- 202 Accepted ------------------|
  |                                    |
  |                                    | [Processing...]
  |                                    |
  |<--- Response (to callback) --------|
  |                                    |
```

**Important Considerations:**

1. **Callback Endpoint Must Be Hosted**: The client must provide a publicly accessible WUS-compliant service endpoint
2. **Same Security Requirements**: The callback must use the same Digikoppeling profile (TLS, WS-Security)
3. **MessageID Correlation**: Use `wsa:RelatesTo` in the response to reference the original `wsa:MessageID`
4. **Timeout Handling**: Need to handle cases where callbacks fail or timeout
5. **Certificate Management**: The service provider must trust the client's callback endpoint certificate

**When to Use Async Callbacks:**

✅ **Good for:**
- Long-running operations (> 30 seconds)
- Heavy processing that can't complete quickly
- Batch operations
- Integration with legacy systems that take time

❌ **Not needed for:**
- Simple queries
- Fast lookups
- Real-time interactions
- When immediate response is required

### MTOM Attachments Support

According to the Digikoppeling specification (section 2.5.3), **MTOM is supported but optional**:

> "In de Digikoppeling Koppelvlakstandaard WUS worden twee mogelijkheden ondersteund om binaire data te versturen. Dat zijn Base64Binary (Base64Binary in payload element van het bericht) of MTOM (MIME wrappers waarbij binaire data in een aparte Multipart/Related pakket is opgenomen)."

Translation: *"In the Digikoppeling WUS Interface Standard, two possibilities are supported for sending binary data: Base64Binary (in the payload element of the message) or MTOM (MIME wrappers where binary data is included in a separate Multipart/Related package)."*

**Key Points on MTOM:**

| Aspect | Details |
|--------|---------|
| **Support Status** | MTOM is fully supported in all WUS profiles |
| **Optional Feature** | Attachments are marked as "Optional" in all three profiles (2W-be, 2W-be-S, 2W-be-SE) |
| **Decision Authority** | **Requirement WM001**: "Toepassen MTOM wordt door webservice provider bepaald" (Application of MTOM is determined by the webservice provider) |
| **Alternatives** | Base64Binary encoding inline in XML is always supported |
| **Implementation** | Most modern toolkits support both sending and receiving MTOM messages |

**Practical Implications:**

1. **Provider Choice**: When setting up a service, the provider decides whether to use MTOM
2. **Consumer Compliance**: Consumers must support whatever the provider chooses (MTOM or Base64Binary)
3. **Existing Services**: When adding a new consumer to an existing service, the consumer must conform to the existing MTOM configuration
4. **Both Formats Valid**: Services can be tested with both optimized (MTOM) and non-optimized formats

**MTOM in Hybrid SOAP Context:**

MTOM itself doesn't add to the SOAP version mixing challenge because:
- MTOM is a packaging mechanism at the HTTP/MIME level
- It works with both SOAP 1.1 and SOAP 1.2
- The SOAP envelope remains unchanged; only the transport packaging differs
- Apache CXF handles MTOM well through its attachment APIs

## The SOAP Version Mixing Problem

### The Hybrid Message Challenge

Unlike a scenario where you need to support either SOAP 1.1 OR SOAP 1.2 messages, WUS requires something more complex: **a single message that combines SOAP 1.1 envelope structure with SOAP 1.2 features**.

### Example of a WUS Hybrid Message

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- SOAP 1.1 Envelope namespace -->
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
               xmlns:wsa="http://www.w3.org/2005/08/addressing"
               xmlns:wsse="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd"
               xmlns:wsu="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd">
    
    <soap:Header>
        <!-- WS-Addressing (SOAP 1.2 feature) in SOAP 1.1 envelope -->
        <wsa:To>http://service.overheid.nl/endpoint?OIN=00000001234567890000</wsa:To>
        <wsa:Action>http://www.example.nl/service/RequestData</wsa:Action>
        <wsa:MessageID>uuid:12345678-1234-1234-1234-123456789012</wsa:MessageID>
        <wsa:ReplyTo>
            <wsa:Address>http://www.w3.org/2005/08/addressing/anonymous</wsa:Address>
        </wsa:ReplyTo>
        
        <!-- WS-Security (1.1 specification, designed for SOAP 1.2 model) -->
        <wsse:Security>
            <wsu:Timestamp>
                <wsu:Created>2025-11-25T10:30:00.000Z</wsu:Created>
                <wsu:Expires>2025-11-25T10:35:00.000Z</wsu:Expires>
            </wsu:Timestamp>
            <wsse:BinarySecurityToken>...</wsse:BinarySecurityToken>
            <ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
                <!-- Signature elements -->
            </ds:Signature>
        </wsse:Security>
    </soap:Header>
    
    <!-- SOAP 1.1 Body -->
    <soap:Body>
        <DataRequest xmlns="http://www.example.nl/schema">
            <!-- Payload -->
        </DataRequest>
    </soap:Body>
</soap:Envelope>
```

### Why This Creates Problems

Standard Java web service frameworks cannot handle this hybrid approach because:

1. **Namespace Incompatibility**: 
   - SOAP 1.1 envelope namespace: `http://schemas.xmlsoap.org/soap/envelope/`
   - WS-Addressing (SOAP 1.2): `http://www.w3.org/2005/08/addressing`
   - Frameworks expect all headers to match the envelope version

2. **Message Factory Lock-in**: 
   - `MessageFactory.newInstance(SOAPConstants.SOAP_1_1_PROTOCOL)` creates a SOAP 1.1-only factory
   - `MessageFactory.newInstance(SOAPConstants.SOAP_1_2_PROTOCOL)` creates a SOAP 1.2-only factory
   - Neither can properly handle the hybrid message structure

3. **Header Processing Conflicts**:
   - SOAP 1.1 header processing rules differ from SOAP 1.2
   - WS-Addressing assumes SOAP 1.2 fault handling
   - WS-Security 1.1 references SOAP 1.2 security constructs

4. **Binding and Protocol Mismatch**:
   - HTTP Content-Type for SOAP 1.1: `text/xml`
   - WS-Addressing expects: `application/soap+xml` (SOAP 1.2)
   - Frameworks enforce strict protocol matching

5. **Standards Compliance Enforcement**:
   - WS-I Basic Profile 1.2 requires SOAP 1.1
   - WS-I BSP 1.1 requires WS-Security 1.1 (which assumes SOAP 1.2 patterns)
   - Frameworks validate against one profile or the other, not both simultaneously

### The Fundamental Issue

The problem isn't that WUS supports two versions - it's that **WUS mandates a non-standard hybrid** that violates the strict versioning assumptions built into standard SOAP frameworks. It's asking for:

```
SOAP 1.1 Envelope 
  + SOAP 1.2 Addressing 
  + SOAP 1.2-style WS-Security 
  + SOAP 1.1 HTTP binding
  = Framework Incompatibility
```

## Affected Java Frameworks

All major Java web service frameworks struggle with WUS's hybrid SOAP approach because they are designed to enforce strict version compliance, not to mix SOAP 1.1 envelopes with SOAP 1.2 features.

### 1. **Apache Axis (1.x)**

- **Status**: Legacy, maintenance mode
- **Issue**: Only supports pure SOAP 1.1; cannot handle WS-Addressing or modern WS-Security
- **WUS Compatibility**: ❌ Not viable
- **Recommendation**: Do not use for new Digikoppeling implementations

### 2. **Apache Axis2**

- **Status**: Mature, in maintenance
- **Issue**: While it supports both SOAP versions separately, it validates envelope/header consistency
  - Cannot mix SOAP 1.1 envelope with SOAP 1.2 addressing headers
  - WS-Addressing module expects matching SOAP version
  - Rampart (WS-Security) validates namespace coherence
- **WUS Compatibility**: ⚠️ Requires significant customization
- **Custom Work Needed**: 
  - Modify AxisEngine to bypass version validation
  - Patch AddressingHandler to accept SOAP 1.1 + WS-Addressing mix
  - Customize Rampart security validators
- **Complexity**: Very High

### 3. **Apache CXF**

- **Status**: Actively maintained
- **Issue**: SoapVersion enum enforces envelope-feature matching
  - SOAP 1.1 binding rejects SOAP 1.2 headers
  - WS-Addressing interceptor validates against binding version
  - WSS4J (WS-Security) expects consistent version throughout message
- **WUS Compatibility**: ⚠️ Best candidate for customization
- **Advantages**:
  - Modular interceptor architecture
  - Clear separation of concerns
  - Excellent WS-Security support via WSS4J
  - Active community and documentation
- **Custom Work Needed**:
  - Create hybrid SoapVersion (e.g., `SOAP_11_WITH_12_FEATURES`)
  - Custom interceptor to relax version validation
  - Modified SoapBindingFactory for hybrid messages
- **Recommendation**: ✅ **Best choice** for WUS implementation

### 4. **Metro (GlassFish/Oracle)**

- **Status**: Actively maintained by Oracle
- **Issue**: WSIT stack enforces strict WS-I compliance
  - SOAPVersion class doesn't support hybrid scenarios
  - JAX-WS runtime validates header/envelope version match
  - Tube pipeline assumes version consistency
- **WUS Compatibility**: ⚠️ Difficult to customize
- **Challenges**:
  - Less modular than CXF
  - Proprietary Oracle extensions
  - Harder to override core validation logic
- **Complexity**: High

### 5. **Spring Web Services (Spring-WS)**

- **Status**: Actively maintained
- **Issue**: Contract-first approach with rigid SOAP version binding
  - WebServiceMessage implementations are version-specific
  - SaajSoapMessage enforces envelope-header coherence
  - WS-Addressing module expects matching SOAP namespace
- **WUS Compatibility**: ⚠️ Very difficult
- **Challenges**:
  - Deeply integrated with SAAJ
  - Message abstraction doesn't anticipate hybrid messages
  - Would require extensive framework modification
- **Complexity**: Very High

### 6. **JBoss Web Services (JBossWS)**

- **Status**: Part of WildFly/JBoss EAP
- **Issue**: Delegates to either CXF or Metro, inheriting their limitations
  - Container-managed, less flexibility for low-level customization
  - Application server integration adds complexity
- **WUS Compatibility**: ⚠️ Depends on underlying stack
- **Recommendation**: Use standalone CXF instead for better control

### Framework Comparison for WUS Implementation

| Framework | WUS Viable | Customization Effort | Maintainability | Recommendation |
|-----------|------------|---------------------|-----------------|----------------|
| Axis 1.x | ❌ No | N/A | Legacy | Avoid |
| Axis2 | ⚠️ Yes | Very High | Medium | Not recommended |
| CXF | ✅ Yes | Medium | High | **Best choice** |
| Metro | ⚠️ Yes | High | Medium | Second choice |
| Spring-WS | ⚠️ Barely | Very High | Low | Not recommended |
| JBossWS | ⚠️ Yes | High | Medium | Use CXF directly instead |

### Why Apache CXF is the Best Choice

1. **Modular Architecture**: Interceptor chain allows targeted modifications
2. **Clean Separation**: SOAP handling, addressing, and security are separate concerns
3. **Extensibility**: Designed to be extended and customized
4. **WSS4J Integration**: Best-in-class WS-Security implementation
5. **Active Development**: Regular updates and good community support
6. **Documentation**: Extensive docs on customization and extension points

## The Shading Solution

### Concept

The challenge of WUS's hybrid SOAP approach requires a specialized framework that:

1. Accepts SOAP 1.1 envelopes as valid
2. Doesn't reject SOAP 1.2 features (WS-Addressing, WS-Security 1.1) within those envelopes
3. Properly processes the mixed namespace structure
4. Maintains WS-I Basic Profile 1.2 and BSP 1.1 compliance

**JAR shading** (package relocation) enables this by allowing you to:

1. Create a **modified copy** of Apache CXF
2. **Customize version validation** to accept hybrid messages
3. **Relocate packages** to avoid classpath conflicts with standard CXF
4. **Use both frameworks** simultaneously - standard CXF for normal services, shaded CXF for Digikoppeling

### Why Shading is Essential

You might ask: "Why not just configure standard CXF differently?" The answer is that the hybrid SOAP requirements violate fundamental assumptions in the framework:

- **Version validation is deeply embedded**: It's not just a configuration option
- **Multiple code points enforce coherence**: Envelope namespace, header validation, binding logic
- **Standard CXF must remain unchanged**: You need it for normal SOAP services
- **Breaking changes would affect all services**: Customization must be isolated

Shading allows you to maintain **two parallel CXF installations**:

```
Standard CXF (org.apache.cxf)
  └─> Normal SOAP 1.2 services
  └─> Internal APIs
  └─> Standard integrations

Shaded CXF (nl.gov.digikoppeling.shaded.cxf)
  └─> Digikoppeling WUS services
  └─> Hybrid SOAP 1.1 + 1.2 features
  └─> PKIoverheid integration
```

### Architecture Overview

```
┌─────────────────────────────────────────────┐
│         Integration Middleware              │
├─────────────────────────────────────────────┤
│                                             │
│  ┌───────────────┐    ┌──────────────────┐ │
│  │  Standard     │    │   Shaded         │ │
│  │  CXF 3.x      │    │   CXF 3.x        │ │
│  │               │    │   (Hybrid SOAP)  │ │
│  │  Package:     │    │   Package:       │ │
│  │  org.apache   │    │   nl.gov.digi    │ │
│  │  .cxf         │    │   koppeling.     │ │
│  │               │    │   shaded.cxf     │ │
│  │               │    │                  │ │
│  │  Pure SOAP    │    │  SOAP 1.1 +      │ │
│  │  1.1 or 1.2   │    │  SOAP 1.2        │ │
│  │               │    │  features        │ │
│  │               │    │  (WUS hybrid)    │ │
│  └───────┬───────┘    └────────┬─────────┘ │
│          │                     │           │
│          │ Standard            │ WUS       │
│          │ Services            │ Services  │
└──────────┼─────────────────────┼───────────┘
           │                     │
           ▼                     ▼
    REST APIs           Digikoppeling WUS
    SOAP Services       Services
    Internal Systems    (Hybrid SOAP)
```

### Implementation Strategy

#### Step 1: Choose Apache CXF as Base Framework

**Why CXF?**

- Clean interceptor-based architecture
- Separate phases for envelope, header, and body processing
- Excellent WSS4J integration for WS-Security
- Extensible without core modifications
- Well-documented extension points

#### Step 2: Create a Shaded Version with Maven

```xml
<project>
    <groupId>nl.gov.digikoppeling</groupId>
    <artifactId>digikoppeling-cxf-wus</artifactId>
    <version>1.0.0</version>
    
    <dependencies>
        <!-- Base CXF dependencies -->
        <dependency>
            <groupId>org.apache.cxf</groupId>
            <artifactId>cxf-rt-frontend-jaxws</artifactId>
            <version>3.6.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.cxf</groupId>
            <artifactId>cxf-rt-ws-security</artifactId>
            <version>3.6.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.cxf</groupId>
            <artifactId>cxf-rt-ws-addr</artifactId>
            <version>3.6.2</version>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.5.1</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <artifactSet>
                                <includes>
                                    <include>org.apache.cxf:*</include>
                                    <include>org.apache.wss4j:*</include>
                                    <include>org.apache.santuario:*</include>
                                </includes>
                            </artifactSet>
                            <relocations>
                                <relocation>
                                    <pattern>org.apache.cxf</pattern>
                                    <shadedPattern>nl.gov.digikoppeling.shaded.cxf</shadedPattern>
                                </relocation>
                                <relocation>
                                    <pattern>org.apache.wss4j</pattern>
                                    <shadedPattern>nl.gov.digikoppeling.shaded.wss4j</shadedPattern>
                                </relocation>
                                <relocation>
                                    <pattern>org.apache.xml.security</pattern>
                                    <shadedPattern>nl.gov.digikoppeling.shaded.xmlsec</shadedPattern>
                                </relocation>
                            </relocations>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                    <resource>META-INF/cxf/bus-extensions.txt</resource>
                                </transformer>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                    <resource>META-INF/cxf/extensions.xml</resource>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

#### Step 3: Create Hybrid SOAP Support Interceptors

The key is to **relax version validation** while maintaining security and correctness.

**Hybrid SOAP Binding Factory:**

```java
package nl.gov.digikoppeling.cxf.wus.binding;

import nl.gov.digikoppeling.shaded.cxf.binding.soap.SoapBindingFactory;
import nl.gov.digikoppeling.shaded.cxf.binding.soap.SoapVersion;
import nl.gov.digikoppeling.shaded.cxf.binding.soap.model.SoapBindingInfo;

/**
 * Custom binding factory that creates hybrid SOAP 1.1/1.2 bindings
 * for Digikoppeling WUS compliance.
 */
public class WusHybridBindingFactory extends SoapBindingFactory {
    
    public static final String WUS_HYBRID_BINDING = 
        "http://schemas.xmlsoap.org/wsdl/soap/digikoppeling-wus";
    
    @Override
    public Binding createBinding(BindingInfo binding) {
        // Create a SOAP 1.1 base binding but with hybrid support
        SoapBindingInfo sbi = (SoapBindingInfo) binding;
        sbi.setSoapVersion(SoapVersion.SOAP_11);
        
        Binding result = super.createBinding(binding);
        
        // Add hybrid message validator that accepts SOAP 1.2 headers
        result.getInInterceptors().add(new HybridSoapVersionInterceptor());
        result.getOutInterceptors().add(new HybridSoapVersionInterceptor());
        
        return result;
    }
}
```

**Hybrid SOAP Version Interceptor:**

```java
package nl.gov.digikoppeling.cxf.wus.interceptor;

import nl.gov.digikoppeling.shaded.cxf.interceptor.Fault;
import nl.gov.digikoppeling.shaded.cxf.message.Message;
import nl.gov.digikoppeling.shaded.cxf.phase.AbstractPhaseInterceptor;
import nl.gov.digikoppeling.shaded.cxf.phase.Phase;
import org.w3c.dom.Element;

import javax.xml.namespace.QName;

/**
 * Interceptor that allows SOAP 1.2 features (WS-Addressing, WS-Security 1.1)
 * in SOAP 1.1 envelopes for Digikoppeling WUS hybrid SOAP support.
 */
public class HybridSoapVersionInterceptor extends AbstractPhaseInterceptor<Message> {
    
    private static final String SOAP11_NS = "http://schemas.xmlsoap.org/soap/envelope/";
    private static final String SOAP12_NS = "http://www.w3.org/2003/05/soap-envelope";
    private static final String WSA_NS = "http://www.w3.org/2005/08/addressing";
    private static final String WSSE_NS = 
        "http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd";
    
    public HybridSoapVersionInterceptor() {
        super(Phase.POST_STREAM);
        addAfter("org.apache.cxf.binding.soap.interceptor.ReadHeadersInterceptor");
    }
    
    @Override
    public void handleMessage(Message message) throws Fault {
        // Get the SOAP headers
        List<Header> headers = CastUtils.cast((List<?>) message.get(Header.HEADER_LIST));
        
        if (headers == null || headers.isEmpty()) {
            return;
        }
        
        // Verify envelope is SOAP 1.1
        String envelopeNS = (String) message.get("soap.envelope.namespace");
        if (!SOAP11_NS.equals(envelopeNS)) {
            throw new Fault(new Exception(
                "Digikoppeling WUS requires SOAP 1.1 envelope, found: " + envelopeNS));
        }
        
        // Allow SOAP 1.2 headers (WS-Addressing, WS-Security 1.1) in SOAP 1.1 envelope
        for (Header header : headers) {
            QName headerQName = header.getName();
            String headerNS = headerQName.getNamespaceURI();
            
            // These SOAP 1.2 namespaces are explicitly allowed in WUS hybrid mode
            if (WSA_NS.equals(headerNS) || WSSE_NS.equals(headerNS)) {
                // Mark as validated even though version doesn't match envelope
                message.getExchange().put("wus.hybrid.header." + headerQName, true);
            }
        }
        
        // Set flag indicating this is a WUS hybrid message
        message.getExchange().put("digikoppeling.wus.hybrid", true);
    }
}
```

**Namespace Validator Override:**

```java
package nl.gov.digikoppeling.cxf.wus.interceptor;

import nl.gov.digikoppeling.shaded.cxf.binding.soap.interceptor.SoapHeaderInterceptor;
import nl.gov.digikoppeling.shaded.cxf.message.Message;

/**
 * Overrides default SOAP header validation to allow WS-Addressing
 * and WS-Security 1.1 headers in SOAP 1.1 envelopes.
 */
public class WusHeaderInterceptor extends SoapHeaderInterceptor {
    
    @Override
    public void handleMessage(Message message) {
        // Check if this is a WUS hybrid message
        Boolean isWusHybrid = (Boolean) message.getExchange()
            .get("digikoppeling.wus.hybrid");
        
        if (Boolean.TRUE.equals(isWusHybrid)) {
            // Disable strict namespace validation for hybrid messages
            message.put("disable.header.namespace.validation", true);
        }
        
        super.handleMessage(message);
    }
}
```

#### Step 4: Configure WS-Security for Hybrid SOAP

WS-Security 1.1 (required by WS-I BSP 1.1) was designed with SOAP 1.2 in mind, but WUS requires it in SOAP 1.1 envelopes. This needs special handling:

```java
package nl.gov.digikoppeling.wus.security;

import nl.gov.digikoppeling.shaded.cxf.ws.security.wss4j.WSS4JOutInterceptor;
import nl.gov.digikoppeling.shaded.cxf.ws.security.wss4j.WSS4JInInterceptor;
import org.apache.wss4j.dom.handler.WSHandlerConstants;

import java.util.HashMap;
import java.util.Map;

/**
 * Configures WS-Security for Digikoppeling WUS hybrid SOAP messages.
 * Uses WS-Security 1.1 (SOAP 1.2 style) in SOAP 1.1 envelopes.
 */
public class WusSecurityConfigurer {
    
    public static void configureOutboundSecurity(JaxWsProxyFactoryBean factory) {
        Map<String, Object> outProps = new HashMap<>();
        
        // Action: Sign and optionally encrypt
        outProps.put(WSHandlerConstants.ACTION, 
            WSHandlerConstants.SIGNATURE + " " + WSHandlerConstants.TIMESTAMP);
        
        // PKIoverheid certificate alias
        outProps.put(WSHandlerConstants.SIGNATURE_USER, "digikoppeling-cert");
        
        // Password callback for private key
        outProps.put(WSHandlerConstants.PW_CALLBACK_CLASS, 
            PKIoverheidPasswordCallback.class.getName());
        
        // Crypto properties for PKIoverheid keystore
        outProps.put(WSHandlerConstants.SIG_PROP_FILE, "pkioverheid-crypto.properties");
        
        // Signature algorithm (SHA-256)
        outProps.put(WSHandlerConstants.SIG_ALGO, 
            "http://www.w3.org/2001/04/xmldsig-more#rsa-sha256");
        outProps.put(WSHandlerConstants.SIG_DIGEST_ALGO, 
            "http://www.w3.org/2001/04/xmlenc#sha256");
        
        // Include certificate in message (DirectReference)
        outProps.put(WSHandlerConstants.SIG_KEY_ID, "DirectReference");
        
        // Parts to sign: Body, WS-Addressing headers, Timestamp
        outProps.put(WSHandlerConstants.SIGNATURE_PARTS, 
            "{Element}{http://schemas.xmlsoap.org/soap/envelope/}Body;" +
            "{Element}{http://www.w3.org/2005/08/addressing}To;" +
            "{Element}{http://www.w3.org/2005/08/addressing}Action;" +
            "{Element}{http://www.w3.org/2005/08/addressing}MessageID;" +
            "{Element}{http://www.w3.org/2005/08/addressing}ReplyTo;" +
            "{Element}{http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd}Timestamp");
        
        // Timestamp configuration
        outProps.put(WSHandlerConstants.TTL_TIMESTAMP, "300"); // 5 minutes
        
        // CRITICAL: Allow WS-Security 1.1 in SOAP 1.1 envelope
        outProps.put("ws-security.enable.soap11.wsse11", "true");
        
        WSS4JOutInterceptor wssOut = new WSS4JOutInterceptor(outProps);
        factory.getOutInterceptors().add(wssOut);
    }
    
    public static void configureInboundSecurity(JaxWsServerFactoryBean factory) {
        Map<String, Object> inProps = new HashMap<>();
        
        // Action: Verify signature and timestamp
        inProps.put(WSHandlerConstants.ACTION,
            WSHandlerConstants.SIGNATURE + " " + WSHandlerConstants.TIMESTAMP);
        
        // Password callback (not used for signature verification but required)
        inProps.put(WSHandlerConstants.PW_CALLBACK_CLASS, 
            PKIoverheidPasswordCallback.class.getName());
        
        // Crypto properties for PKIoverheid truststore
        inProps.put(WSHandlerConstants.SIG_VER_PROP_FILE, 
            "pkioverheid-truststore.properties");
        
        // Enable BSP compliance (but with hybrid SOAP allowance)
        inProps.put(WSHandlerConstants.ENABLE_BSP_COMPLIANCE, "true");
        inProps.put("ws-security.validate.soap11.with.wsse11", "true");
        
        // Timestamp validation
        inProps.put(WSHandlerConstants.TTL_TIMESTAMP, "300");
        inProps.put(WSHandlerConstants.TTL_FUTURE_TIMESTAMP, "60"); // Clock skew
        
        // Require signature
        inProps.put(WSHandlerConstants.REQUIRE_SIGNED_ENCRYPTED_DATA_ELEMENTS, "true");
        
        WSS4JInInterceptor wssIn = new WSS4JInInterceptor(inProps);
        factory.getInInterceptors().add(wssIn);
    }
    
    /**
     * For encrypted messages (profile Digikoppeling 2W-be-SE)
     */
    public static void addEncryption(Map<String, Object> props, String recipientAlias) {
        // Add encryption action
        String existingAction = (String) props.get(WSHandlerConstants.ACTION);
        props.put(WSHandlerConstants.ACTION, 
            existingAction + " " + WSHandlerConstants.ENCRYPT);
        
        // Recipient's public key
        props.put(WSHandlerConstants.ENCRYPTION_USER, recipientAlias);
        props.put(WSHandlerConstants.ENC_PROP_FILE, "pkioverheid-crypto.properties");
        
        // Encryption algorithm
        props.put(WSHandlerConstants.ENC_SYM_ALGO, 
            "http://www.w3.org/2009/xmlenc11#aes256-gcm");
        props.put(WSHandlerConstants.ENC_KEY_TRANSPORT, 
            "http://www.w3.org/2001/04/xmlenc#rsa-oaep-mgf1p");
        
        // Encrypt SOAP Body only (not WS-Addressing headers)
        props.put(WSHandlerConstants.ENCRYPTION_PARTS,
            "{Content}{http://schemas.xmlsoap.org/soap/envelope/}Body");
    }
}
```

**PKIoverheid Crypto Properties:**

```properties
# pkioverheid-crypto.properties
org.apache.ws.security.crypto.provider=org.apache.wss4j.common.crypto.Merlin

# Keystore for own certificate and private key
org.apache.ws.security.crypto.merlin.keystore.type=pkcs12
org.apache.ws.security.crypto.merlin.keystore.file=/path/to/pkioverheid-keystore.p12
org.apache.ws.security.crypto.merlin.keystore.password=changeit
org.apache.ws.security.crypto.merlin.keystore.alias=digikoppeling-cert

# Truststore for validating other parties' certificates
org.apache.ws.security.crypto.merlin.truststore.type=JKS
org.apache.ws.security.crypto.merlin.truststore.file=/path/to/pkioverheid-truststore.jks
org.apache.ws.security.crypto.merlin.truststore.password=changeit

# Load CA certificates (PKIoverheid root and intermediate)
org.apache.ws.security.crypto.merlin.load.cacerts=true
```

#### Step 5: Configure MTOM Support (Optional)

If the service provider has decided to use MTOM for attachments, configuration is straightforward:

**Java Example:**

```java
package nl.gov.digikoppeling.wus.mtom;

import nl.gov.digikoppeling.shaded.cxf.jaxws.JaxWsProxyFactoryBean;
import nl.gov.digikoppeling.shaded.cxf.jaxws.JaxWsServerFactoryBean;

public class MtomConfiguration {
    
    /**
     * Enable MTOM on client side
     */
    public static void enableMtomClient(JaxWsProxyFactoryBean factory) {
        // Enable MTOM
        Map<String, Object> props = new HashMap<>();
        props.put("mtom-enabled", Boolean.TRUE);
        factory.setProperties(props);
        
        // Alternative: Set on endpoint directly
        // factory.getProperties().put("mtom-enabled", true);
    }
    
    /**
     * Enable MTOM on server side
     */
    public static void enableMtomServer(JaxWsServerFactoryBean factory) {
        // Enable MTOM
        Map<String, Object> props = new HashMap<>();
        props.put("mtom-enabled", Boolean.TRUE);
        factory.setProperties(props);
    }
    
    /**
     * Example service method with attachment
     */
    @WebService
    public interface DocumentService {
        
        @WebMethod
        Response uploadDocument(
            @WebParam(name = "metadata") DocumentMetadata metadata,
            @WebParam(name = "attachment") DataHandler attachment
        );
    }
    
    /**
     * Implementation handling MTOM attachment
     */
    @WebService(endpointInterface = "nl.gov.digikoppeling.wus.mtom.DocumentService")
    public class DocumentServiceImpl implements DocumentService {
        
        @Override
        public Response uploadDocument(DocumentMetadata metadata, DataHandler attachment) {
            try {
                // Process the attachment
                InputStream is = attachment.getInputStream();
                String contentType = attachment.getContentType();
                String fileName = attachment.getName();
                
                // Handle the binary data
                byte[] content = IOUtils.toByteArray(is);
                
                // Process...
                return new Response("Document processed successfully");
                
            } catch (IOException e) {
                throw new RuntimeException("Failed to process attachment", e);
            }
        }
    }
}
```

**Kotlin Example:**

```kotlin
package nl.gov.digikoppeling.wus.mtom

import nl.gov.digikoppeling.shaded.cxf.jaxws.JaxWsProxyFactoryBean
import nl.gov.digikoppeling.shaded.cxf.jaxws.JaxWsServerFactoryBean
import javax.activation.DataHandler
import javax.jws.WebMethod
import javax.jws.WebParam
import javax.jws.WebService

/**
 * Enable MTOM with Kotlin DSL style
 */
fun JaxWsProxyFactoryBean.enableMtom() {
    properties = mutableMapOf("mtom-enabled" to true)
}

fun JaxWsServerFactoryBean.enableMtom() {
    properties = mutableMapOf("mtom-enabled" to true)
}

/**
 * Service interface with attachment support
 */
@WebService
interface DocumentService {
    
    @WebMethod
    fun uploadDocument(
        @WebParam(name = "metadata") metadata: DocumentMetadata,
        @WebParam(name = "attachment") attachment: DataHandler
    ): Response
}

/**
 * Implementation with Kotlin-style attachment handling
 */
@WebService(endpointInterface = "nl.gov.digikoppeling.wus.mtom.DocumentService")
class DocumentServiceImpl : DocumentService {
    
    override fun uploadDocument(
        metadata: DocumentMetadata, 
        attachment: DataHandler
    ): Response = runCatching {
        attachment.inputStream.use { stream ->
            val content = stream.readBytes()
            val contentType = attachment.contentType
            val fileName = attachment.name
            
            // Process the binary data
            processDocument(content, metadata)
            
            Response("Document processed successfully")
        }
    }.getOrElse { ex ->
        throw RuntimeException("Failed to process attachment", ex)
    }
    
    private fun processDocument(content: ByteArray, metadata: DocumentMetadata) {
        // Business logic
    }
}

/**
 * Client usage example
 */
class DocumentClient(private val serviceUrl: String) {
    
    private val service: DocumentService by lazy {
        JaxWsProxyFactoryBean().apply {
            serviceClass = DocumentService::class.java
            address = serviceUrl
            enableMtom()
            
            // Configure WUS hybrid SOAP
            inInterceptors.add(HybridSoapVersionInterceptor())
            WusSecurityConfigurer.configureOutboundSecurity(this)
        }.create() as DocumentService
    }
    
    fun uploadFile(file: File, metadata: DocumentMetadata): Response {
        val dataHandler = DataHandler(FileDataSource(file))
        return service.uploadDocument(metadata, dataHandler)
    }
}
```

**MTOM with WS-Security Considerations:**

When combining MTOM with WS-Security (profiles 2W-be-S and 2W-be-SE):

```java
// MTOM attachments are handled separately from the SOAP body
// Encryption applies only to the SOAP body, not MTOM parts by default

// To encrypt MTOM attachments as well (if required):
outProps.put(WSHandlerConstants.ENCRYPTION_PARTS,
    "{Content}{http://schemas.xmlsoap.org/soap/envelope/}Body;" +
    "{Content}{http://www.w3.org/2005/05/xmlmime}Include"); // Encrypt attachment references

// Signing MTOM attachments:
outProps.put(WSHandlerConstants.SIGNATURE_PARTS,
    "{Element}{http://schemas.xmlsoap.org/soap/envelope/}Body;" +
    "{Element}{http://www.w3.org/2005/08/addressing}Action;" +
    // ... other WS-Addressing headers
    "{Element}{http://www.w3.org/2005/05/xmlmime}Include"); // Sign attachment references
```

**Note on MTOM and Hybrid SOAP:**

MTOM works seamlessly with the hybrid SOAP approach because:
- MTOM operates at the HTTP/MIME packaging level
- It doesn't affect the SOAP envelope structure
- Both SOAP 1.1 and 1.2 support MTOM equally
- The hybrid SOAP headers remain in the SOAP envelope; only binary data is externalized

### WUS Profiles Summary

The Digikoppeling WUS standard defines three profiles, all of which can optionally use MTOM:

| Profile Name | Profile Code | 2-way TLS | Signed | Encrypted | MTOM Attachments |
|--------------|--------------|-----------|--------|-----------|------------------|
| Best Effort | Digikoppeling 2W-be | ✅ Required | ❌ No | ❌ No | ⚠️ Optional |
| Best Effort – Signed | Digikoppeling 2W-be-S | ✅ Required | ✅ Required | ❌ No | ⚠️ Optional |
| Best Effort – Signed & Encrypted | Digikoppeling 2W-be-SE | ✅ Required | ✅ Required | ✅ Required | ⚠️ Optional |

**Profile Characteristics:**

- **2W-be (Best Effort)**: 
  - Simplest profile
  - Only transport-level security (TLS)
  - No message-level security
  - Suitable for non-sensitive data or where TLS is sufficient

- **2W-be-S (Signed)**:
  - Adds message-level authentication and integrity
  - Signs SOAP Body, WS-Addressing headers, and Timestamp
  - Enables non-repudiation
  - Certificate included in message (DirectReference)

- **2W-be-SE (Signed & Encrypted)**:
  - Highest security level
  - Signs and encrypts SOAP Body
  - WS-Addressing headers remain unencrypted (needed for routing)
  - Sign-then-encrypt pattern

**Critical Implementation Notes:**

1. **Hybrid SOAP is always present**: All profiles use SOAP 1.1 envelope with SOAP 1.2 features
2. **MTOM is provider-determined**: Consumers must support both MTOM and Base64Binary
3. **Security is profile-level**: All operations in a service use the same profile
4. **PKIoverheid certificates required**: For profiles with signing/encryption
5. **Async callbacks use WS-Addressing**: `wsa:ReplyTo` can specify callback endpoints for asynchronous patterns

### Implementing Asynchronous Callbacks

**Java Example - Client Making Async Request:**

```java
package nl.gov.digikoppeling.wus.async;

import nl.gov.digikoppeling.shaded.cxf.jaxws.JaxWsProxyFactoryBean;
import org.apache.cxf.ws.addressing.WSAddressingFeature;
import org.apache.cxf.ws.addressing.EndpointReferenceType;

public class AsyncCallbackClient {
    
    public void makeAsyncRequest(String serviceUrl, String callbackUrl) {
        JaxWsProxyFactoryBean factory = new JaxWsProxyFactoryBean();
        factory.setServiceClass(DigikoppelingService.class);
        factory.setAddress(serviceUrl);
        
        // Enable WS-Addressing
        WSAddressingFeature addressingFeature = new WSAddressingFeature();
        addressingFeature.setAddressingRequired(true);
        factory.getFeatures().add(addressingFeature);
        
        // Configure hybrid SOAP
        factory.getInInterceptors().add(new HybridSoapVersionInterceptor());
        
        // Configure WS-Security
        WusSecurityConfigurer.configureOutboundSecurity(factory);
        
        DigikoppelingService service = (DigikoppelingService) factory.create();
        
        // Set the callback endpoint in WS-Addressing headers
        Map<String, Object> requestContext = 
            ((BindingProvider) service).getRequestContext();
        
        // Create ReplyTo endpoint reference
        EndpointReferenceType replyTo = new EndpointReferenceType();
        AttributedURIType address = new AttributedURIType();
        address.setValue(callbackUrl + "?OIN=00000009876543210000");
        replyTo.setAddress(address);
        
        requestContext.put("javax.xml.ws.addressing.replyTo", replyTo);
        
        // Set MessageID for correlation
        String messageId = "uuid:" + UUID.randomUUID().toString();
        requestContext.put("javax.xml.ws.addressing.messageId", messageId);
        
        // Store messageId for later correlation
        storeMessageIdForCallback(messageId);
        
        try {
            // Make async request - expects 202 Accepted or similar
            service.processAsyncRequest(request);
            System.out.println("Async request sent with MessageID: " + messageId);
        } catch (Exception e) {
            // Handle synchronous errors
            System.err.println("Failed to send async request: " + e.getMessage());
        }
    }
    
    private void storeMessageIdForCallback(String messageId) {
        // Store in database or cache for correlation when callback arrives
    }
}
```

**Java Example - Service Hosting Callback Endpoint:**

```java
package nl.gov.digikoppeling.wus.async;

import nl.gov.digikoppeling.shaded.cxf.jaxws.JaxWsServerFactoryBean;

@WebService
public class CallbackServiceImpl {
    
    @WebMethod
    public void handleAsyncResponse(
        @WebParam(name = "response") ResponseType response,
        @WebParam(header = true, 
                  name = "RelatesTo", 
                  targetNamespace = "http://www.w3.org/2005/08/addressing")
        String relatesTo
    ) {
        // Correlate with original request using RelatesTo (= original MessageID)
        String originalMessageId = relatesTo;
        
        // Retrieve original request context
        OriginalRequest originalRequest = retrieveByMessageId(originalMessageId);
        
        if (originalRequest == null) {
            throw new RuntimeException("Unknown MessageID: " + originalMessageId);
        }
        
        // Process the async response
        processAsyncResponse(originalRequest, response);
    }
    
    private OriginalRequest retrieveByMessageId(String messageId) {
        // Retrieve from database or cache
        return null; // Implement
    }
    
    private void processAsyncResponse(
        OriginalRequest request, 
        ResponseType response
    ) {
        // Business logic to handle the callback
    }
}

/**
 * Configure and publish the callback endpoint
 */
public class CallbackEndpointPublisher {
    
    public static void publishCallbackEndpoint(String callbackUrl) {
        JaxWsServerFactoryBean factory = new JaxWsServerFactoryBean();
        factory.setServiceClass(CallbackServiceImpl.class);
        factory.setAddress(callbackUrl);
        
        // Configure hybrid SOAP support
        factory.getInInterceptors().add(new HybridSoapVersionInterceptor());
        
        // Configure WS-Security for incoming callbacks
        WusSecurityConfigurer.configureInboundSecurity(factory);
        
        // Enable WS-Addressing
        WSAddressingFeature addressingFeature = new WSAddressingFeature();
        addressingFeature.setAddressingRequired(true);
        factory.getFeatures().add(addressingFeature);
        
        // Publish the endpoint
        factory.create();
        
        System.out.println("Callback endpoint published at: " + callbackUrl);
    }
}
```

**Kotlin Example - Async Client with DSL:**

```kotlin
package nl.gov.digikoppeling.wus.async

import nl.gov.digikoppeling.shaded.cxf.jaxws.JaxWsProxyFactoryBean
import org.apache.cxf.ws.addressing.WSAddressingFeature
import org.apache.cxf.ws.addressing.EndpointReferenceType
import org.apache.cxf.ws.addressing.AttributedURIType
import java.util.UUID
import javax.xml.ws.BindingProvider

class AsyncCallbackClient(
    private val serviceUrl: String,
    private val callbackUrl: String
) {
    private val service: DigikoppelingService by lazy {
        JaxWsProxyFactoryBean().apply {
            serviceClass = DigikoppelingService::class.java
            address = serviceUrl
            
            // Enable WS-Addressing
            features.add(WSAddressingFeature().apply {
                isAddressingRequired = true
            })
            
            // Configure hybrid SOAP
            inInterceptors.add(HybridSoapVersionInterceptor())
            
            // Configure WS-Security
            WusSecurityConfigurer.configureOutboundSecurity(this)
        }.create() as DigikoppelingService
    }
    
    suspend fun makeAsyncRequest(request: RequestType): String = withContext(Dispatchers.IO) {
        val messageId = "uuid:${UUID.randomUUID()}"
        
        // Configure WS-Addressing headers for async
        (service as BindingProvider).requestContext.apply {
            // Set ReplyTo with actual callback endpoint
            put("javax.xml.ws.addressing.replyTo", createReplyToEndpoint())
            
            // Set MessageID for correlation
            put("javax.xml.ws.addressing.messageId", messageId)
        }
        
        // Store for later correlation
        storeMessageIdForCallback(messageId, request)
        
        try {
            // Send async request
            service.processAsyncRequest(request)
            println("Async request sent with MessageID: $messageId")
            messageId
        } catch (e: Exception) {
            error("Failed to send async request: ${e.message}")
        }
    }
    
    private fun createReplyToEndpoint(): EndpointReferenceType {
        return EndpointReferenceType().apply {
            address = AttributedURIType().apply {
                value = "$callbackUrl?OIN=00000009876543210000"
            }
        }
    }
    
    private suspend fun storeMessageIdForCallback(messageId: String, request: RequestType) {
        // Store in database/cache for correlation
    }
}

/**
 * Callback service implementation with Kotlin coroutines
 */
@WebService
class AsyncCallbackService(
    private val responseProcessor: suspend (String, ResponseType) -> Unit
) {
    
    @WebMethod
    fun handleAsyncResponse(
        @WebParam(name = "response") response: ResponseType,
        @WebParam(
            header = true,
            name = "RelatesTo",
            targetNamespace = "http://www.w3.org/2005/08/addressing"
        ) relatesTo: String
    ) {
        // Launch coroutine for async processing
        GlobalScope.launch {
            try {
                responseProcessor(relatesTo, response)
            } catch (e: Exception) {
                println("Error processing async response: ${e.message}")
            }
        }
    }
}

/**
 * DSL for publishing callback endpoint
 */
fun publishCallbackEndpoint(
    url: String,
    block: AsyncCallbackEndpointConfig.() -> Unit
) {
    val config = AsyncCallbackEndpointConfig().apply(block)
    
    JaxWsServerFactoryBean().apply {
        serviceClass = AsyncCallbackService::class.java
        address = url
        
        // Configure hybrid SOAP
        inInterceptors.add(HybridSoapVersionInterceptor())
        
        // Configure WS-Security
        WusSecurityConfigurer.configureInboundSecurity(this)
        
        // Enable WS-Addressing
        features.add(WSAddressingFeature().apply {
            isAddressingRequired = true
        })
        
        serviceBean = AsyncCallbackService(config.responseHandler)
    }.create()
    
    println("Callback endpoint published at: $url")
}

data class AsyncCallbackEndpointConfig(
    var responseHandler: suspend (messageId: String, response: ResponseType) -> Unit = { _, _ -> }
)

/**
 * Usage example
 */
fun main() {
    // Publish callback endpoint
    publishCallbackEndpoint("https://client.overheid.nl/callback") {
        responseHandler = { messageId, response ->
            println("Received async response for $messageId")
            // Process response
        }
    }
    
    // Make async request
    val client = AsyncCallbackClient(
        serviceUrl = "https://service.overheid.nl/wus",
        callbackUrl = "https://client.overheid.nl/callback"
    )
    
    runBlocking {
        val messageId = client.makeAsyncRequest(RequestType())
        println("Waiting for callback for $messageId...")
    }
}
```

**Key Implementation Points for Async Callbacks:**

1. **Both endpoints need Digikoppeling compliance**: The callback endpoint must also implement hybrid SOAP + WS-Security
2. **Certificate trust**: The service provider must trust the client's callback endpoint certificate
3. **Firewall configuration**: Callback endpoints must be publicly accessible
4. **Correlation mechanism**: Store `MessageID` to match with `RelatesTo` in callback
5. **Timeout handling**: Implement timeout logic for callbacks that never arrive
6. **Idempotency**: Handle duplicate callbacks gracefully

## Integration with Middleware

### Enterprise Service Bus (ESB) Integration

When integrating with middleware platforms, the shaded framework approach allows:

#### 1. **Apache Camel**

```kotlin
// Kotlin DSL for Camel route
from("direct:digikoppeling-input")
    .process { exchange ->
        // Use shaded CXF for Digikoppeling
        val endpoint = nl.gov.digikoppeling.shaded.cxf.jaxws.JaxWsProxyFactoryBean()
        endpoint.serviceClass = DigikoppelingService::class.java
        endpoint.address = "https://service.overheid.nl/wus"
        
        val service = endpoint.create() as DigikoppelingService
        val response = service.processRequest(exchange.`in`.getBody(Request::class.java))
        exchange.message.body = response
    }
    .to("direct:standard-processing")

from("direct:standard-service")
    .to("cxf:bean:standardCxfEndpoint") // Uses regular CXF
```

#### 2. **MuleSoft**

```xml
<!-- Custom connector using shaded framework -->
<digikoppeling:config name="Digikoppeling_Config">
    <digikoppeling:connection 
        keystoreFile="${keystore.path}"
        keystorePassword="${keystore.password}"
        truststore="${truststore.path}" />
</digikoppeling:config>

<flow name="digikoppelingFlow">
    <digikoppeling:invoke 
        config-ref="Digikoppeling_Config"
        service="StufBerichtService"
        operation="beantwoordVraag"/>
</flow>
```

#### 3. **Spring Integration**

```kotlin
@Configuration
class DigikoppelingIntegrationConfig {
    
    @Bean
    fun digikoppelingGateway(): IntegrationFlow {
        return IntegrationFlows
            .from("digikoppelingInput")
            .handle { payload, headers ->
                // Use shaded framework
                val factory = nl.gov.digikoppeling.shaded.cxf.jaxws.JaxWsServerFactoryBean()
                configureDualSoapSupport(factory)
                factory.create()
            }
            .get()
    }
    
    @Bean
    fun standardGateway(): IntegrationFlow {
        return IntegrationFlows
            .from("standardInput")
            .handle { payload, headers ->
                // Use standard CXF
                val factory = org.apache.cxf.jaxws.JaxWsServerFactoryBean()
                configureStandardSoap(factory)
                factory.create()
            }
            .get()
    }
}
```

### Benefits in Middleware Context

1. **Isolation**: Shaded framework doesn't interfere with standard middleware operations
2. **Coexistence**: Run Digikoppeling-specific and standard services side-by-side
3. **Maintainability**: Update standard framework without affecting custom Digikoppeling implementation
4. **Testing**: Test both implementations independently

## Java and Kotlin Considerations

### Java Implementation

**Advantages:**
- Wider ecosystem support
- Most web service frameworks are Java-native
- Extensive documentation and examples
- Better tooling for WSDL generation and consumption

**Example Service Implementation:**

```java
package nl.gov.organization.digikoppeling;

import javax.jws.WebService;
import javax.jws.WebMethod;
import javax.jws.soap.SOAPBinding;

@WebService(
    targetNamespace = "http://www.example.nl/digikoppeling/service",
    serviceName = "DigikoppelingService"
)
@SOAPBinding(style = SOAPBinding.Style.DOCUMENT)
public class DigikoppelingServiceImpl {
    
    @WebMethod
    public ResponseType processRequest(RequestType request) {
        // Business logic
        return processBusinessLogic(request);
    }
    
    private ResponseType processBusinessLogic(RequestType request) {
        // Implementation
    }
}
```

**Service Factory Configuration:**

```java
package nl.gov.organization.digikoppeling.config;

import nl.gov.digikoppeling.shaded.cxf.jaxws.JaxWsServerFactoryBean;

public class DigikoppelingServiceFactory {
    
    public static void publishService() {
        JaxWsServerFactoryBean factory = new JaxWsServerFactoryBean();
        factory.setServiceClass(DigikoppelingServiceImpl.class);
        factory.setAddress("https://service.example.nl/digikoppeling/wus");
        
        // Add dual SOAP version support
        factory.getInInterceptors().add(new DualSoapVersionInterceptor());
        
        // Configure WS-Security
        configureWsSecurity(factory);
        
        // Create and publish
        factory.create();
    }
    
    private static void configureWsSecurity(JaxWsServerFactoryBean factory) {
        Map<String, Object> inProps = new HashMap<>();
        inProps.put(WSHandlerConstants.ACTION, 
                   WSHandlerConstants.SIGNATURE + " " + WSHandlerConstants.ENCRYPT);
        inProps.put(WSHandlerConstants.PW_CALLBACK_CLASS, 
                   PKIoverheidCallbackHandler.class.getName());
        
        WSS4JInInterceptor wssIn = new WSS4JInInterceptor(inProps);
        factory.getInInterceptors().add(wssIn);
    }
}
```

### Kotlin Implementation

**Advantages:**
- Modern language features (null safety, coroutines, extension functions)
- Better DSL support for configuration
- Improved readability
- Seamless Java interoperability
- Growing adoption in enterprise

**Example Service Implementation:**

```kotlin
package nl.gov.organization.digikoppeling

import javax.jws.WebService
import javax.jws.WebMethod
import javax.jws.soap.SOAPBinding

@WebService(
    targetNamespace = "http://www.example.nl/digikoppeling/service",
    serviceName = "DigikoppelingService"
)
@SOAPBinding(style = SOAPBinding.Style.DOCUMENT)
class DigikoppelingServiceImpl {
    
    @WebMethod
    fun processRequest(request: RequestType): ResponseType {
        return processBusinessLogic(request)
    }
    
    private fun processBusinessLogic(request: RequestType): ResponseType {
        // Implementation with Kotlin features
        return request.let { req ->
            // Null-safe processing
            ResponseType().apply {
                status = processStatus(req.status)
                data = req.data?.let { transformData(it) }
            }
        }
    }
}
```

**Service Factory Configuration (Kotlin DSL):**

```kotlin
package nl.gov.organization.digikoppeling.config

import nl.gov.digikoppeling.shaded.cxf.jaxws.JaxWsServerFactoryBean

object DigikoppelingServiceFactory {
    
    fun publishService() {
        val factory = JaxWsServerFactoryBean().apply {
            serviceClass = DigikoppelingServiceImpl::class.java
            address = "https://service.example.nl/digikoppeling/wus"
            
            // Add dual SOAP version support
            inInterceptors.add(DualSoapVersionInterceptor())
            
            // Configure WS-Security
            configureWsSecurity(this)
        }
        
        factory.create()
    }
    
    private fun configureWsSecurity(factory: JaxWsServerFactoryBean) {
        val inProps = mapOf(
            WSHandlerConstants.ACTION to "${WSHandlerConstants.SIGNATURE} ${WSHandlerConstants.ENCRYPT}",
            WSHandlerConstants.PW_CALLBACK_CLASS to PKIoverheidCallbackHandler::class.java.name
        )
        
        factory.inInterceptors.add(WSS4JInInterceptor(inProps))
    }
}
```

**Kotlin DSL for Configuration:**

```kotlin
// Extension function for fluent configuration
fun JaxWsServerFactoryBean.configureDualSoapSupport(
    block: DualSoapConfig.() -> Unit
) {
    val config = DualSoapConfig().apply(block)
    
    // Apply dual SOAP interceptors
    inInterceptors.add(DualSoapVersionInterceptor(config))
    outInterceptors.add(DualSoapResponseInterceptor(config))
}

// DSL usage
val factory = JaxWsServerFactoryBean().apply {
    serviceClass = DigikoppelingServiceImpl::class.java
    address = "https://service.example.nl/digikoppeling/wus"
    
    configureDualSoapSupport {
        defaultVersion = SoapVersion.SOAP_12
        allowSoap11 = true
        allowSoap12 = true
        autoDetect = true
    }
    
    configureWsSecurity {
        signatureAlgorithm = "http://www.w3.org/2001/04/xmldsig-more#rsa-sha256"
        digestAlgorithm = "http://www.w3.org/2001/04/xmlenc#sha256"
        certificateProvider = PKIoverheidCertificateProvider()
    }
}
```

**Coroutine Support for Async Operations:**

```kotlin
import kotlinx.coroutines.*

class AsyncDigikoppelingClient(
    private val endpoint: String
) {
    private val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())
    
    suspend fun processRequestAsync(request: RequestType): ResponseType = 
        withContext(Dispatchers.IO) {
            val factory = nl.gov.digikoppeling.shaded.cxf.jaxws.JaxWsProxyFactoryBean()
            factory.serviceClass = DigikoppelingService::class.java
            factory.address = endpoint
            
            val service = factory.create() as DigikoppelingService
            service.processRequest(request)
        }
    
    suspend fun batchProcess(requests: List<RequestType>): List<ResponseType> = 
        coroutineScope {
            requests.map { request ->
                async { processRequestAsync(request) }
            }.awaitAll()
        }
}
```

### Language Choice Recommendation

**Use Java if:**
- Team is more familiar with Java
- Existing codebase is Java-based
- Maximum compatibility with legacy tools is required
- Enterprise mandates Java

**Use Kotlin if:**
- Starting a new project
- Team is open to modern languages
- Want better DSL capabilities for configuration
- Need async/reactive patterns (coroutines)
- Want improved null safety
- Enterprise is polyglot-friendly

**Hybrid Approach:**
- Core Digikoppeling adapter in Java (for maximum compatibility)
- Application logic and integration in Kotlin (for productivity)
- Both can seamlessly interoperate

## Practical Implementation

### Project Structure

```
digikoppeling-adapter/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── nl/gov/digikoppeling/
│   │   │       ├── adapter/
│   │   │       │   ├── DigikoppelingServiceAdapter.java
│   │   │       │   └── WsSecurityConfigurer.java
│   │   │       ├── interceptor/
│   │   │       │   ├── DualSoapVersionInterceptor.java
│   │   │       │   └── SoapVersionDetector.java
│   │   │       └── security/
│   │   │           └── PKIoverheidCallbackHandler.java
│   │   ├── kotlin/
│   │   │   └── nl/gov/digikoppeling/
│   │   │       ├── config/
│   │   │       │   └── DigikoppelingConfiguration.kt
│   │   │       └── dsl/
│   │   │           └── ServiceConfigDsl.kt
│   │   └── resources/
│   │       ├── crypto.properties
│   │       └── cxf-custom-extensions.xml
│   └── test/
│       ├── java/
│       └── kotlin/
└── shaded-cxf/
    └── pom.xml (shading configuration)
```

### Testing Strategy

#### Unit Tests

```kotlin
class DualSoapVersionInterceptorTest {
    
    private lateinit var interceptor: DualSoapVersionInterceptor
    private lateinit var message: Message
    
    @BeforeEach
    fun setup() {
        interceptor = DualSoapVersionInterceptor()
        message = mock()
    }
    
    @Test
    fun `should detect SOAP 1_1 from namespace`() {
        val soap11Envelope = """
            <?xml version="1.0"?>
            <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
                <soap:Body>
                    <test/>
                </soap:Body>
            </soap:Envelope>
        """.trimIndent()
        
        val version = interceptor.detectVersion(soap11Envelope.byteInputStream())
        assertEquals(SoapVersion.SOAP_11, version)
    }
    
    @Test
    fun `should detect SOAP 1_2 from namespace`() {
        val soap12Envelope = """
            <?xml version="1.0"?>
            <env:Envelope xmlns:env="http://www.w3.org/2003/05/soap-envelope">
                <env:Body>
                    <test/>
                </env:Body>
            </env:Envelope>
        """.trimIndent()
        
        val version = interceptor.detectVersion(soap12Envelope.byteInputStream())
        assertEquals(SoapVersion.SOAP_12, version)
    }
}
```

#### Integration Tests

```kotlin
@SpringBootTest
@TestPropertySource(locations = ["classpath:test.properties"])
class DigikoppelingIntegrationTest {
    
    @Autowired
    private lateinit var serviceAdapter: DigikoppelingServiceAdapter
    
    @Test
    fun `should accept SOAP 1_1 request and respond correctly`() {
        val request = createSoap11Request()
        val response = serviceAdapter.processRequest(request)
        
        assertNotNull(response)
        assertTrue(response.isSoap11Compatible())
    }
    
    @Test
    fun `should accept SOAP 1_2 request and respond correctly`() {
        val request = createSoap12Request()
        val response = serviceAdapter.processRequest(request)
        
        assertNotNull(response)
        assertTrue(response.isSoap12Compatible())
    }
    
    @Test
    fun `should validate WS-Security signature`() {
        val signedRequest = createSignedSoap12Request()
        val response = serviceAdapter.processRequest(signedRequest)
        
        assertNotNull(response)
        assertTrue(response.isSignatureValid())
    }
}
```

### Performance Considerations

1. **Connection Pooling**: Configure HTTP client connection pools
2. **Message Caching**: Cache parsed SOAP messages where appropriate
3. **Lazy Initialization**: Delay factory creation until first use
4. **Thread Safety**: Ensure interceptors are thread-safe
5. **Memory Management**: Be mindful of large message handling

```kotlin
@Configuration
class PerformanceConfiguration {
    
    @Bean
    fun httpConduitConfigurer(): HTTPConduitConfigurer {
        return HTTPConduitConfigurer { httpConduit ->
            httpConduit.client.apply {
                connectionTimeout = 30000
                receiveTimeout = 60000
                allowChunking = true
                autoRedirect = false
            }
        }
    }
    
    @Bean
    fun messageCacheInterceptor(): MessageCacheInterceptor {
        return MessageCacheInterceptor().apply {
            maxCacheSize = 100
            timeToLive = 300000 // 5 minutes
        }
    }
}
```

## Recommendations

### Short-term (0-6 months)

1. **Assess Current State**
   - Audit existing services and their SOAP versions
   - Identify Digikoppeling WUS requirements
   - Evaluate current framework usage (Axis, CXF, etc.)

2. **Proof of Concept**
   - Build a shaded CXF module
   - Implement dual SOAP version detection
   - Test with sample Digikoppeling services

3. **Create Adapter Layer**
   - Develop reusable Digikoppeling adapter
   - Package as internal library
   - Document usage patterns

### Medium-term (6-12 months)

1. **Migrate Critical Services**
   - Start with high-priority Digikoppeling integrations
   - Implement phased rollout
   - Monitor production behavior

2. **Enhance Tooling**
   - Create Maven archetypes for new services
   - Develop testing utilities
   - Build monitoring dashboards

3. **Knowledge Transfer**
   - Train development teams
   - Create internal documentation
   - Establish best practices

### Long-term (12+ months)

1. **Standardize Approach**
   - Make shaded adapter the standard for Digikoppeling
   - Deprecate old Axis-based implementations
   - Consolidate on single framework approach

2. **Contribute Back**
   - Consider open-sourcing the adapter (if allowed)
   - Contribute improvements to Apache CXF
   - Share knowledge with Dutch government community

3. **Monitor Evolution**
   - Track Digikoppeling standard updates
   - Stay current with CXF releases
   - Plan for future SOAP 1.1 deprecation

### Best Practices

1. **Version Control**
   - Tag shaded framework versions clearly
   - Document changes to custom interceptors
   - Maintain compatibility matrix

2. **Testing**
   - Test against both SOAP versions
   - Validate WS-Security implementation
   - Use Digikoppeling compliance test suite

3. **Security**
   - Regularly update PKIoverheid certificates
   - Keep WS-Security libraries current
   - Audit security configurations

4. **Documentation**
   - Maintain architecture decision records (ADRs)
   - Document deviation from standards
   - Keep runbooks updated

5. **Monitoring**
   - Log SOAP version for each request
   - Monitor WS-Security validation failures
   - Track performance metrics

## Conclusion

The Digikoppeling WUS profile presents a unique technical challenge: it mandates a **hybrid SOAP approach** that mixes SOAP 1.1 envelope structure with SOAP 1.2 features (WS-Addressing and WS-Security 1.1). This is fundamentally different from supporting either SOAP 1.1 or SOAP 1.2 separately - it requires processing messages that violate the strict version segregation built into standard frameworks.

### The Core Challenge

```
WUS Requirement = SOAP 1.1 Envelope + SOAP 1.2 Features
```

This combination creates messages that:
- Use SOAP 1.1 namespaces for the envelope
- Include WS-Addressing (a SOAP 1.2 specification) in the header
- Apply WS-Security 1.1 (designed for SOAP 1.2 security model)
- Follow WS-I Basic Profile 1.2 (SOAP 1.1) and BSP 1.1 (SOAP 1.2 security) simultaneously

### Why Standard Frameworks Fail

Standard Java web service frameworks (Apache Axis, CXF, Metro, Spring-WS, JBossWS) enforce **version coherence** - they expect all parts of a SOAP message to use the same version. This is a fundamental design assumption that WUS violates by design.

### The Shading Solution

By creating a **customized, shaded version of Apache CXF**, organizations can:

1. **Maintain Standards Compliance**: Implement WUS's hybrid SOAP requirement fully
2. **Preserve Framework Isolation**: Keep standard CXF for normal SOAP services
3. **Enable Coexistence**: Run both standard and Digikoppeling WUS services in the same middleware
4. **Ensure Maintainability**: Isolate WUS-specific customizations from mainstream code

The shaded adapter becomes a specialized component that understands the WUS hybrid SOAP model, while the rest of the architecture remains standard and portable.

### Key Technical Insights

1. **It's Not About Supporting Two Versions**: WUS doesn't ask you to support SOAP 1.1 *or* SOAP 1.2 - it requires mixing them in single messages

2. **Validation Must Be Relaxed Strategically**: The solution isn't to disable all validation, but to allow specific SOAP 1.2 namespaces (WS-Addressing, WS-Security 1.1) within SOAP 1.1 envelopes

3. **Shading Enables Parallel Frameworks**: You need both standard-compliant and WUS-compliant SOAP stacks running simultaneously

4. **PKIoverheid Integration is Critical**: WS-Security configuration must work with Dutch government PKI certificates

### Implementation Path

Whether implementing in Java or Kotlin, the recommended approach is:

1. **Start with Apache CXF**: Best architecture for this type of customization
2. **Create a Shaded Version**: Use Maven Shade Plugin to relocate packages
3. **Add Hybrid SOAP Interceptors**: Relax version validation for WUS messages
4. **Configure WS-Security Carefully**: Handle WS-Security 1.1 in SOAP 1.1 envelopes
5. **Test Thoroughly**: Validate against Digikoppeling compliance test suite

### Looking Forward

As the Dutch government continues its digital transformation journey, understanding and correctly implementing the WUS hybrid SOAP approach will be essential for public sector organizations. The investment in a well-architected, shaded framework adapter provides:

- **Immediate Compliance**: Meet Digikoppeling WUS requirements today
- **Long-term Flexibility**: Easily update standard frameworks without affecting WUS implementation
- **Reduced Integration Complexity**: Single adapter handles all WUS peculiarities
- **Improved Interoperability**: Correctly process the hybrid SOAP messages that other Dutch government systems send

The hybrid SOAP challenge is real, but with the right architecture (shading), the right base framework (CXF), and the right understanding of the WUS specification, it's entirely solvable. The key is recognizing that WUS isn't asking for version flexibility - it's defining a specific hybrid that requires targeted framework customization.

---

**Key Takeaway**: Digikoppeling WUS doesn't support both SOAP 1.1 and 1.2 - it **mixes** them in a single hybrid standard. This requires a specialized framework approach, and JAR shading with Apache CXF customization is the most maintainable solution for Java/Kotlin implementations.

---

## Quick Reference: WUS Feature Summary

| Feature | Status | Description | Impact on Implementation |
|---------|--------|-------------|-------------------------|
| **Hybrid SOAP** | ✅ Required | SOAP 1.1 envelope + SOAP 1.2 features (WS-Addressing, WS-Security 1.1) | 🔴 **High** - Requires custom framework |
| **WS-Addressing** | ✅ Required | Headers: To, Action, MessageID, RelatesTo, ReplyTo, From | 🟡 **Medium** - Must handle in hybrid context |
| **WS-Security 1.0/1.1** | ⚠️ Profile-dependent | Both namespaces used (BSP 1.1) | 🟡 **Medium** - Configure for SOAP 1.1 envelope |
| **PKIoverheid Certificates** | ✅ Required | X.509 certificates for signing/encryption | 🟡 **Medium** - Standard PKI handling |
| **Two-way TLS** | ✅ Required | Mutual TLS authentication | 🟢 **Low** - Standard TLS config |
| **MTOM Attachments** | ⚠️ Optional | Provider decides, consumer must support | 🟢 **Low** - Standard MTOM support |
| **Async Callbacks** | ⚠️ Optional | WS-Addressing ReplyTo with actual endpoint | 🟡 **Medium** - Requires callback hosting |
| **OIN in URLs** | ⚠️ Optional | Organization ID as query parameter | 🟢 **Low** - URL construction |
| **SignatureConfirmation** | ⚠️ If WS-Security | WS-Security 1.1 feature | 🟢 **Low** - WSS4J handles it |

**Legend:**
- ✅ Required = Must be implemented
- ⚠️ Optional/Conditional = Depends on profile or provider
- 🔴 High = Major framework modification needed
- 🟡 Medium = Configuration and custom code needed
- 🟢 Low = Standard implementation sufficient

---

---

## Additional Resources

### Official Documentation
- [Digikoppeling Standards](https://www.logius.nl/domeinen/gegevensuitwisseling/digikoppeling)
- [PKIoverheid Certificates](https://www.logius.nl/diensten/pkioverheid)
- [Apache CXF Documentation](https://cxf.apache.org/docs/)

### Tools
- [SoapUI](https://www.soapui.org/) - For testing SOAP services
- [Digikoppeling Compliance Validator](https://www.logius.nl/compliance-voorziening)

### Community
- [Pleio Digikoppeling Community](https://pleio.nl/groups/digikoppeling)
- [Apache CXF Mailing Lists](https://cxf.apache.org/mailing-lists.html)

---

**Document Version**: 1.0  
**Last Updated**: November 2025  
**Author**: Technical Architecture Team
