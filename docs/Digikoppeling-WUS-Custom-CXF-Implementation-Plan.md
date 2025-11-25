# Digikoppeling WUS Custom CXF Implementation Plan

## Project: Shaded Apache CXF for Digikoppeling WUS Hybrid SOAP Support

**Version**: 1.0  
**Date**: November 2025  
**Status**: Planning Phase
**Author**: Ir. Marcel A. Grauwen

---

## Executive Summary

This document outlines a comprehensive plan to develop a customized, shaded Apache CXF framework that supports Digikoppeling WUS's unique hybrid SOAP requirement (SOAP 1.1 envelope with SOAP 1.2 features). The project will deliver a production-ready library that enables Java/Kotlin applications to interact with Dutch government services via Digikoppeling WUS without compromising standard SOAP implementations.

**Key Deliverables**:
- Shaded CXF JAR with hybrid SOAP support
- WUS-specific interceptors and handlers
- Comprehensive test suite
- Documentation and examples
- Compliance validation against Digikoppeling test suite

**Estimated Duration**: 12-16 weeks  
**Team Size**: 2-3 developers  
**Language**: Java 11+ with optional Kotlin support

---

## Table of Contents

1. [Project Objectives](#project-objectives)
2. [Technical Decisions](#technical-decisions)
3. [CXF Version Selection](#cxf-version-selection)
4. [Shading Strategy](#shading-strategy)
5. [Customization Approach](#customization-approach)
6. [Testing Strategy](#testing-strategy)
7. [Development Phases](#development-phases)
8. [Team and Skills](#team-and-skills)
9. [Risk Assessment](#risk-assessment)
10. [Deliverables](#deliverables)
11. [Success Criteria](#success-criteria)

---

## Project Objectives

### Primary Objectives

1. **Enable Hybrid SOAP Processing**
   - Accept SOAP 1.1 envelopes with SOAP 1.2 WS-Addressing headers
   - Support WS-Security 1.0 and 1.1 namespaces in SOAP 1.1 context
   - Maintain WS-I Basic Profile 1.2 and BSP 1.1 compliance

2. **Ensure Isolation**
   - Package relocation to avoid conflicts with standard CXF
   - Allow coexistence with standard CXF in same application
   - Zero impact on existing non-Digikoppeling services

3. **Production-Ready Quality**
   - Pass Digikoppeling Compliance Test Suite
   - Handle all three WUS profiles (2W-be, 2W-be-S, 2W-be-SE)
   - Support MTOM and async callback patterns
   - Comprehensive error handling

4. **Developer-Friendly**
   - Clear documentation and examples
   - Simple configuration API
   - Good error messages
   - Maven Central deployment (or internal repository)

### Non-Objectives

- ❌ Not creating a fork of CXF (we're customizing via shading)
- ❌ Not modifying CXF source code directly
- ❌ Not supporting ebMS2 or REST-API Digikoppeling profiles
- ❌ Not replacing all SOAP frameworks (only for WUS use cases)

---

## Technical Decisions

### Decision Matrix

| Decision Point | Choice | Rationale |
|----------------|--------|-----------|
| **Base Framework** | Apache CXF 3.6.x | Most recent stable, best WS-Security support, modular architecture |
| **Java Version** | Java 11 (target), Java 17 (development) | Balance between compatibility and modern features |
| **Build Tool** | Maven 3.9+ | Industry standard, excellent shade plugin support |
| **Primary Language** | Java | Maximum compatibility, easier debugging of framework issues |
| **Kotlin Support** | Yes (separate module) | Optional Kotlin DSL wrapper around Java core |
| **Testing Framework** | JUnit 5 + TestContainers | Modern testing, can spin up full WUS endpoints |
| **Shading Tool** | Maven Shade Plugin 3.5+ | Mature, well-documented, handles complex relocations |
| **CI/CD** | GitHub Actions or GitLab CI | Automated builds, testing, compliance validation |

### Why Java as Primary Language

**Advantages**:
- CXF is Java-native; staying in Java reduces complexity
- Easier to debug framework-level issues
- Better stack traces when things go wrong
- Simpler Maven/Gradle integration
- Wider community support for Java + CXF

**Kotlin Support Strategy**:
- Core framework in Java
- Optional Kotlin wrapper module with DSL
- Kotlin consumers can still use Java core directly
- Best of both worlds approach

---

## CXF Version Selection

### Version Analysis

| CXF Version | Release Date | Java Requirement | Recommendation |
|-------------|--------------|------------------|----------------|
| **3.6.4** | Oct 2024 | Java 11+ | ✅ **Recommended** - Latest stable |
| 3.5.9 | Sep 2024 | Java 8+ | ⚠️ Fallback if Java 8 needed |
| 3.4.x | EOL | Java 8+ | ❌ Avoid - End of life |
| 4.0.x | Beta | Java 11+ | ❌ Too new - not stable yet |

### Selected Version: Apache CXF 3.6.4

**Justification**:
1. **Mature and Stable**: Production-ready, well-tested
2. **Active Maintenance**: Regular security updates
3. **Java 11 Baseline**: Modern features without bleeding edge
4. **Excellent WSS4J Integration**: Version 2.4.x with good WS-Security 1.1 support
5. **Jakarta EE Transition**: Handles both javax.* and jakarta.* namespaces
6. **Good Documentation**: Extensive guides and examples

### Key Dependencies

```xml
<properties>
    <cxf.version>3.6.4</cxf.version>
    <wss4j.version>2.4.3</wss4j.version>
    <xmlsec.version>2.3.4</xmlsec.version>
    <spring.version>5.3.31</spring.version>
</properties>

<dependencies>
    <!-- Core CXF -->
    <dependency>
        <groupId>org.apache.cxf</groupId>
        <artifactId>cxf-rt-frontend-jaxws</artifactId>
        <version>${cxf.version}</version>
    </dependency>
    
    <!-- WS-Security -->
    <dependency>
        <groupId>org.apache.cxf</groupId>
        <artifactId>cxf-rt-ws-security</artifactId>
        <version>${cxf.version}</version>
    </dependency>
    
    <!-- WS-Addressing -->
    <dependency>
        <groupId>org.apache.cxf</groupId>
        <artifactId>cxf-rt-ws-addr</artifactId>
        <version>${cxf.version}</version>
    </dependency>
    
    <!-- MTOM Support -->
    <dependency>
        <groupId>org.apache.cxf</groupId>
        <artifactId>cxf-rt-bindings-soap</artifactId>
        <version>${cxf.version}</version>
    </dependency>
</dependencies>
```

---

## Shading Strategy

### Shading Approach

**Objective**: Relocate CXF packages to avoid conflicts while maintaining functionality.

### Package Relocation Scheme

```
Original Package                          → Shaded Package
────────────────────────────────────────────────────────────────────
org.apache.cxf                           → nl.gov.digikoppeling.shaded.cxf
org.apache.wss4j                         → nl.gov.digikoppeling.shaded.wss4j
org.apache.xml.security                  → nl.gov.digikoppeling.shaded.xmlsec
org.apache.neethi                        → nl.gov.digikoppeling.shaded.neethi

Keep unchanged (common dependencies):
────────────────────────────────────────
javax.xml.* / jakarta.xml.*              → NO RELOCATION
javax.jws.* / jakarta.jws.*              → NO RELOCATION
javax.activation.*                       → NO RELOCATION
org.slf4j.*                              → NO RELOCATION
```

### Maven Shade Plugin Configuration

**Phase 1: Basic Shading Setup**

```xml
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
                <!-- Create sources JAR for debugging -->
                <createSourcesJar>true</createSourcesJar>
                
                <!-- Shade but don't minimize (we need all classes) -->
                <minimizeJar>false</minimizeJar>
                
                <!-- Artifact naming -->
                <shadedArtifactAttached>false</shadedArtifactAttached>
                <finalName>digikoppeling-wus-cxf-${project.version}</finalName>
                
                <!-- What to include -->
                <artifactSet>
                    <includes>
                        <include>org.apache.cxf:*</include>
                        <include>org.apache.wss4j:*</include>
                        <include>org.apache.santuario:xmlsec</include>
                        <include>org.apache.neethi:neethi</include>
                    </includes>
                    <excludes>
                        <!-- Exclude test artifacts -->
                        <exclude>org.apache.cxf:cxf-testutils</exclude>
                    </excludes>
                </artifactSet>
                
                <!-- Package relocations -->
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
                    <relocation>
                        <pattern>org.apache.neethi</pattern>
                        <shadedPattern>nl.gov.digikoppeling.shaded.neethi</shadedPattern>
                    </relocation>
                </relocations>
                
                <!-- Service resource transformers -->
                <transformers>
                    <!-- Merge service descriptors -->
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                    
                    <!-- Append CXF bus extensions -->
                    <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                        <resource>META-INF/cxf/bus-extensions.txt</resource>
                    </transformer>
                    
                    <!-- Append CXF extensions -->
                    <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                        <resource>META-INF/cxf/extensions.xml</resource>
                    </transformer>
                    
                    <!-- Handle Spring handlers -->
                    <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                        <resource>META-INF/spring.handlers</resource>
                    </transformer>
                    
                    <!-- Handle Spring schemas -->
                    <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                        <resource>META-INF/spring.schemas</resource>
                    </transformer>
                    
                    <!-- License information -->
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ApacheLicenseResourceTransformer"/>
                </transformers>
                
                <!-- Filters -->
                <filters>
                    <filter>
                        <artifact>*:*</artifact>
                        <excludes>
                            <!-- Remove signatures -->
                            <exclude>META-INF/*.SF</exclude>
                            <exclude>META-INF/*.DSA</exclude>
                            <exclude>META-INF/*.RSA</exclude>
                            <!-- Remove unnecessary metadata -->
                            <exclude>META-INF/MANIFEST.MF</exclude>
                        </excludes>
                    </filter>
                </filters>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### Validation Steps After Shading

**Automated Checks**:

1. **Class Relocation Verification**
   ```bash
   # Check that packages are relocated
   jar tf target/digikoppeling-wus-cxf-1.0.0.jar | grep "org/apache/cxf"
   # Should return NO results
   
   jar tf target/digikoppeling-wus-cxf-1.0.0.jar | grep "nl/gov/digikoppeling/shaded/cxf"
   # Should return relocated classes
   ```

2. **Service Descriptor Check**
   ```bash
   # Verify META-INF/services files are merged
   jar xf target/digikoppeling-wus-cxf-1.0.0.jar META-INF/services
   cat META-INF/services/org.apache.cxf.bus.factory.BusFactory
   # Should reference shaded classes
   ```

3. **Dependency Analysis**
   ```bash
   mvn dependency:tree
   # Verify no transitive dependencies leak
   ```

### Shading Pitfalls to Avoid

| Issue | Problem | Solution |
|-------|---------|----------|
| **String literals** | Code with `"org.apache.cxf.Xyz"` strings won't be relocated | Use class references: `Xyz.class.getName()` |
| **Reflection** | `Class.forName("org.apache.cxf...")` breaks | Provide configuration to list classes |
| **Spring XML** | Bean class names in XML aren't relocated | Use XmlRootElementTransformer |
| **Service discovery** | SPI files not merged properly | Use ServicesResourceTransformer |
| **Duplicate resources** | Multiple JARs with same resource paths | Use AppendingTransformer |

---

## Customization Approach

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│           Digikoppeling WUS CXF (Shaded)                │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌───────────────────────────────────────────────┐     │
│  │   Custom WUS Components (Java)                │     │
│  ├───────────────────────────────────────────────┤     │
│  │  • WusHybridSoapVersionInterceptor            │     │
│  │  • WusAddressingValidator                     │     │
│  │  • WusSecurityValidator                       │     │
│  │  • WusBindingFactory                          │     │
│  │  • WusServiceFactory                          │     │
│  └───────────────────────────────────────────────┘     │
│                      ▼                                  │
│  ┌───────────────────────────────────────────────┐     │
│  │   Shaded Apache CXF Core                      │     │
│  │   (nl.gov.digikoppeling.shaded.cxf.*)         │     │
│  └───────────────────────────────────────────────┘     │
│                      ▼                                  │
│  ┌───────────────────────────────────────────────┐     │
│  │   Shaded WSS4J & XML Security                 │     │
│  │   (nl.gov.digikoppeling.shaded.wss4j.*)       │     │
│  └───────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### Core Customizations Required

#### 1. Hybrid SOAP Version Interceptor

**File**: `nl/gov/digikoppeling/wus/interceptor/WusHybridSoapVersionInterceptor.java`

**Purpose**: Relax SOAP version validation to allow SOAP 1.2 headers in SOAP 1.1 envelope

**Key Tasks**:
- Detect SOAP envelope version (must be 1.1)
- Allow specific SOAP 1.2 namespaces (WS-Addressing, WS-Security 1.1)
- Set flags for downstream interceptors
- Provide clear error messages if wrong version detected

**Implementation Priority**: 🔴 Critical - Phase 1

**Complexity**: Medium

```java
package nl.gov.digikoppeling.wus.interceptor;

import nl.gov.digikoppeling.shaded.cxf.interceptor.Fault;
import nl.gov.digikoppeling.shaded.cxf.message.Message;
import nl.gov.digikoppeling.shaded.cxf.phase.AbstractPhaseInterceptor;
import nl.gov.digikoppeling.shaded.cxf.phase.Phase;

public class WusHybridSoapVersionInterceptor extends AbstractPhaseInterceptor<Message> {
    
    private static final String SOAP11_NS = "http://schemas.xmlsoap.org/soap/envelope/";
    private static final String WSA_NS = "http://www.w3.org/2005/08/addressing";
    
    public WusHybridSoapVersionInterceptor() {
        super(Phase.POST_STREAM);
        addAfter("org.apache.cxf.binding.soap.interceptor.ReadHeadersInterceptor");
    }
    
    @Override
    public void handleMessage(Message message) throws Fault {
        // Implementation details...
    }
}
```

#### 2. WUS Addressing Validator

**File**: `nl/gov/digikoppeling/wus/addressing/WusAddressingValidator.java`

**Purpose**: Validate WS-Addressing headers according to WUS requirements

**Key Tasks**:
- Verify mandatory headers: To, Action, MessageID
- Validate ReplyTo (anonymous for sync, endpoint for async)
- Validate From (optional OIN in query string)
- Ensure RelatesTo in response matches request MessageID

**Implementation Priority**: 🔴 Critical - Phase 1

**Complexity**: Medium

#### 3. WUS Security Configurator

**File**: `nl/gov/digikoppeling/wus/security/WusSecurityConfigurator.java`

**Purpose**: Configure WS-Security for WUS profiles with hybrid SOAP support

**Key Tasks**:
- Configure signing for profile 2W-be-S
- Configure encryption for profile 2W-be-SE
- Handle PKIoverheid certificate loading
- Ensure WS-Security 1.1 features work in SOAP 1.1 envelope
- Implement SignatureConfirmation (WS-Security 1.1)

**Implementation Priority**: 🟡 High - Phase 2

**Complexity**: High

#### 4. WUS Binding Factory

**File**: `nl/gov/digikoppeling/wus/binding/WusBindingFactory.java`

**Purpose**: Create SOAP bindings that support hybrid version

**Key Tasks**:
- Extend SoapBindingFactory
- Create custom SOAP binding info
- Register WUS-specific interceptors
- Handle both MTOM and non-MTOM configurations

**Implementation Priority**: 🟡 High - Phase 1

**Complexity**: Medium

#### 5. WUS Service Factories

**Files**: 
- `nl/gov/digikoppeling/wus/jaxws/WusJaxWsProxyFactoryBean.java` (client)
- `nl/gov/digikoppeling/wus/jaxws/WusJaxWsServerFactoryBean.java` (server)

**Purpose**: Easy-to-use factories for creating WUS-compliant services

**Key Tasks**:
- Wrap standard CXF factories
- Auto-configure WUS interceptors
- Provide fluent API for configuration
- Support all three WUS profiles

**Implementation Priority**: 🟢 Medium - Phase 2

**Complexity**: Low

### Project Module Structure

```
digikoppeling-wus-cxf/
├── pom.xml (parent)
├── digikoppeling-wus-cxf-core/
│   ├── pom.xml (shading configuration)
│   └── src/
│       ├── main/java/
│       │   └── nl/gov/digikoppeling/wus/
│       │       ├── interceptor/
│       │       │   ├── WusHybridSoapVersionInterceptor.java
│       │       │   └── WusNamespaceInterceptor.java
│       │       ├── addressing/
│       │       │   ├── WusAddressingValidator.java
│       │       │   └── WusAddressingHandler.java
│       │       ├── security/
│       │       │   ├── WusSecurityConfigurator.java
│       │       │   └── PKIoverheidCryptoProvider.java
│       │       ├── binding/
│       │       │   └── WusBindingFactory.java
│       │       ├── jaxws/
│       │       │   ├── WusJaxWsProxyFactoryBean.java
│       │       │   └── WusJaxWsServerFactoryBean.java
│       │       └── WusProfile.java (enum: BE, BE_S, BE_SE)
│       └── main/resources/
│           └── META-INF/
│               └── cxf/
│                   └── bus-extensions-wus.txt
├── digikoppeling-wus-cxf-kotlin/
│   ├── pom.xml (depends on core)
│   └── src/
│       └── main/kotlin/
│           └── nl/gov/digikoppeling/wus/kotlin/
│               ├── WusDsl.kt
│               └── WusExtensions.kt
├── digikoppeling-wus-cxf-spring/
│   ├── pom.xml (Spring integration)
│   └── src/
│       └── main/java/
│           └── nl/gov/digikoppeling/wus/spring/
│               ├── WusNamespaceHandler.java
│               └── WusServiceFactoryBean.java
└── digikoppeling-wus-cxf-examples/
    ├── pom.xml
    └── src/
        └── main/java/
            └── nl/gov/digikoppeling/examples/
                ├── SimpleWusClient.java
                ├── SimpleWusService.java
                ├── SecureWusClient.java
                └── AsyncCallbackExample.java
```

---

## Testing Strategy

### Multi-Level Testing Approach

```
┌─────────────────────────────────────────────┐
│  Level 5: Digikoppeling Compliance Suite   │ ← Final validation
├─────────────────────────────────────────────┤
│  Level 4: Integration Tests                │ ← Full WUS scenarios
├─────────────────────────────────────────────┤
│  Level 3: Component Tests                  │ ← Interceptors, validators
├─────────────────────────────────────────────┤
│  Level 2: Shading Verification             │ ← Package relocation
├─────────────────────────────────────────────┤
│  Level 1: Unit Tests                       │ ← Individual classes
└─────────────────────────────────────────────┘
```

### Level 1: Unit Tests

**Framework**: JUnit 5 + Mockito  
**Coverage Target**: 80%+  
**Execution**: Every build

**Test Categories**:

```java
// Example: Test hybrid SOAP validation
@Test
void shouldAcceptSoap11EnvelopeWithSoap12Addressing() {
    // Given: SOAP 1.1 envelope
    SoapMessage message = createSoap11Message();
    
    // When: Add WS-Addressing (SOAP 1.2) headers
    addWsAddressingHeaders(message);
    
    // Then: Should not throw exception
    assertDoesNotThrow(() -> 
        interceptor.handleMessage(message)
    );
}

@Test
void shouldRejectSoap12Envelope() {
    // Given: SOAP 1.2 envelope
    SoapMessage message = createSoap12Message();
    
    // When: Process with WUS interceptor
    // Then: Should throw clear error
    Fault fault = assertThrows(Fault.class, () ->
        interceptor.handleMessage(message)
    );
    assertThat(fault.getMessage())
        .contains("WUS requires SOAP 1.1 envelope");
}
```

### Level 2: Shading Verification Tests

**Framework**: Custom test harness  
**Execution**: After package phase

**Test Script**: `verify-shading.sh`

```bash
#!/bin/bash
# Verify package relocation

JAR_FILE="target/digikoppeling-wus-cxf-1.0.0.jar"

echo "Checking for unrelocated packages..."

# Should find NO org.apache.cxf classes
if jar tf "$JAR_FILE" | grep -q "org/apache/cxf/"; then
    echo "❌ FAIL: Found unrelocated org.apache.cxf classes"
    exit 1
fi

# Should find shaded classes
if ! jar tf "$JAR_FILE" | grep -q "nl/gov/digikoppeling/shaded/cxf/"; then
    echo "❌ FAIL: Missing shaded classes"
    exit 1
fi

# Verify service descriptors reference shaded classes
jar xf "$JAR_FILE" META-INF/services
for service in META-INF/services/*; do
    if grep -q "org\.apache\.cxf\." "$service"; then
        echo "❌ FAIL: Service descriptor contains unrelocated references: $service"
        exit 1
    fi
done

echo "✅ PASS: Shading verification successful"
```

### Level 3: Component Tests

**Framework**: JUnit 5 + TestContainers  
**Coverage Target**: All major components  
**Execution**: Integration test phase

**Test Setup**:

```java
@Testcontainers
class WusServiceComponentTest {
    
    @Container
    static GenericContainer<?> mockWusService = 
        new GenericContainer<>("mock-wus-service:latest")
            .withExposedPorts(8080)
            .withEnv("DIGIKOPPELING_PROFILE", "2W-be-S");
    
    @Test
    void shouldHandleHybridSoapRequest() {
        // Given: WUS client factory
        WusJaxWsProxyFactoryBean factory = new WusJaxWsProxyFactoryBean();
        factory.setServiceClass(TestService.class);
        factory.setAddress(getServiceUrl());
        factory.setProfile(WusProfile.BE_SIGNED);
        
        TestService service = factory.create(TestService.class);
        
        // When: Make request
        Response response = service.testOperation(new Request());
        
        // Then: Verify hybrid SOAP was accepted
        assertNotNull(response);
    }
}
```

### Level 4: Integration Tests

**Framework**: JUnit 5 + Embedded Jetty + WSS4J  
**Scenarios**: All WUS profiles and patterns

**Test Matrix**:

| Profile | MTOM | Async | Encryption | Test Class |
|---------|------|-------|------------|------------|
| 2W-be | No | No | No | `WusBestEffortIT` |
| 2W-be | Yes | No | No | `WusBestEffortMtomIT` |
| 2W-be | No | Yes | No | `WusBestEffortAsyncIT` |
| 2W-be-S | No | No | No | `WusSignedIT` |
| 2W-be-S | Yes | No | No | `WusSignedMtomIT` |
| 2W-be-S | No | Yes | No | `WusSignedAsyncIT` |
| 2W-be-SE | No | No | Yes | `WusEncryptedIT` |
| 2W-be-SE | Yes | No | Yes | `WusEncryptedMtomIT` |

**Example Integration Test**:

```java
@SpringBootTest
@TestPropertySource("classpath:integration-test.properties")
class WusSignedIntegrationTest {
    
    private WusJaxWsServerFactoryBean serverFactory;
    private WusJaxWsProxyFactoryBean clientFactory;
    
    @BeforeEach
    void setupServiceAndClient() {
        // Server setup
        serverFactory = new WusJaxWsServerFactoryBean();
        serverFactory.setServiceClass(TestServiceImpl.class);
        serverFactory.setAddress("http://localhost:9000/test");
        serverFactory.setProfile(WusProfile.BE_SIGNED);
        serverFactory.setKeystore("classpath:test-keystore.p12");
        serverFactory.setTruststore("classpath:test-truststore.jks");
        serverFactory.create();
        
        // Client setup
        clientFactory = new WusJaxWsProxyFactoryBean();
        clientFactory.setServiceClass(TestService.class);
        clientFactory.setAddress("http://localhost:9000/test");
        clientFactory.setProfile(WusProfile.BE_SIGNED);
        clientFactory.setKeystore("classpath:test-keystore.p12");
        clientFactory.setTruststore("classpath:test-truststore.jks");
    }
    
    @Test
    void shouldExchangeSignedMessages() {
        // Given
        TestService client = clientFactory.create(TestService.class);
        Request request = new Request("test data");
        
        // When
        Response response = client.process(request);
        
        // Then
        assertNotNull(response);
        assertEquals("processed: test data", response.getData());
        
        // Verify WS-Addressing headers were present
        verifyWsAddressingHeaders();
        
        // Verify signature was validated
        verifySignatureValidation();
    }
    
    @Test
    void shouldHandleHybridSoapEnvelope() {
        // Capture actual SOAP message
        MessageCapture capture = new MessageCapture();
        clientFactory.getOutInterceptors().add(capture);
        
        TestService client = clientFactory.create(TestService.class);
        client.process(new Request());
        
        String soapMessage = capture.getCapturedMessage();
        
        // Verify SOAP 1.1 envelope
        assertThat(soapMessage).contains(
            "http://schemas.xmlsoap.org/soap/envelope/"
        );
        
        // Verify SOAP 1.2 WS-Addressing
        assertThat(soapMessage).contains(
            "http://www.w3.org/2005/08/addressing"
        );
    }
}
```

### Level 5: Digikoppeling Compliance Suite

**Framework**: Digikoppeling Compliance Voorziening  
**Source**: https://gitlab.com/logius/digikoppeling-compliance  
**Execution**: Final validation before release

**Compliance Test Categories**:

1. **Basic Profile Compliance**
   - SOAP 1.1 envelope validation
   - Document/literal binding
   - WSDL 1.1 compliance

2. **WS-Addressing Compliance**
   - Mandatory headers present
   - MessageID format (UUID)
   - RelatesTo correlation
   - OIN in URL support

3. **WS-Security Compliance**
   - Timestamp validation
   - Signature validation (SHA-256)
   - Encryption (AES-256-GCM)
   - PKIoverheid certificate chain
   - SignatureConfirmation

4. **Profile-Specific Tests**
   - 2W-be: TLS only
   - 2W-be-S: Signing
   - 2W-be-SE: Signing + Encryption

**Integration Strategy**:

```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>exec-maven-plugin</artifactId>
    <executions>
        <execution>
            <id>digikoppeling-compliance</id>
            <phase>integration-test</phase>
            <goals>
                <goal>exec</goal>
            </goals>
            <configuration>
                <executable>./run-compliance-tests.sh</executable>
                <arguments>
                    <argument>${project.version}</argument>
                    <argument>all-profiles</argument>
                </arguments>
            </configuration>
        </execution>
    </executions>
</plugin>
```

**Compliance Test Script**: `run-compliance-tests.sh`

```bash
#!/bin/bash
# Run Digikoppeling compliance tests

VERSION=$1
PROFILE=$2

# Start test service
java -jar target/test-service-${VERSION}.jar &
SERVICE_PID=$!

# Wait for service to start
sleep 5

# Run compliance tests
docker run --network host \
    logius/digikoppeling-compliance:latest \
    --endpoint http://localhost:8080/wus \
    --profile ${PROFILE} \
    --report-dir ./compliance-reports

# Capture exit code
COMPLIANCE_RESULT=$?

# Stop service
kill $SERVICE_PID

# Exit with compliance result
exit $COMPLIANCE_RESULT
```

### Test Data Management

**PKIoverheid Test Certificates**:

```
src/test/resources/
├── certificates/
│   ├── test-server-keystore.p12      # Test server certificate
│   ├── test-client-keystore.p12      # Test client certificate
│   ├── test-truststore.jks           # CA + intermediate certs
│   └── README.md                     # How to generate/renew
├── wsdl/
│   ├── test-service.wsdl             # Test WSDL
│   └── test-schema.xsd               # Test XSD
└── messages/
    ├── soap11-with-wsa.xml           # Sample hybrid message
    ├── signed-message.xml            # With WS-Security signature
    └── encrypted-message.xml         # With WS-Security encryption
```

### Continuous Testing

**GitHub Actions Workflow**: `.github/workflows/test.yml`

```yaml
name: Test Suite

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Unit Tests
        run: mvn test
      
  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Integration Tests
        run: mvn verify
      
  compliance-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Build
        run: mvn package
      - name: Compliance Tests
        run: ./run-compliance-tests.sh ${{ github.sha }} all-profiles
      - name: Upload Reports
        uses: actions/upload-artifact@v3
        with:
          name: compliance-reports
          path: ./compliance-reports/
```

---

## Development Phases

### Phase 0: Project Setup (Week 1)

**Objectives**:
- Set up project structure
- Configure Maven multi-module build
- Set up CI/CD pipeline
- Establish coding standards

**Deliverables**:
- ✅ Git repository with branch strategy
- ✅ Maven parent POM with dependency management
- ✅ Module structure (core, kotlin, spring, examples)
- ✅ GitHub Actions or GitLab CI configuration
- ✅ Code quality tools (Checkstyle, SpotBugs, SonarQube)

**Tasks**:
1. Create Git repository (GitHub/GitLab)
2. Set up Maven parent POM
3. Configure Maven Shade Plugin (basic)
4. Set up CI/CD pipeline
5. Document development workflow

**Success Criteria**:
- Maven build succeeds
- CI pipeline runs on commit
- Can create shaded JAR (even if empty)

### Phase 1: Core Shading & Basic Hybrid SOAP (Weeks 2-4)

**Objectives**:
- Complete shading configuration
- Implement hybrid SOAP version support
- Basic WS-Addressing support

**Deliverables**:
- ✅ Fully shaded CXF JAR with relocated packages
- ✅ WusHybridSoapVersionInterceptor
- ✅ WusAddressingValidator
- ✅ WusBindingFactory
- ✅ Basic unit tests (60%+ coverage)

**Tasks**:

**Week 2: Shading**
1. Complete Maven Shade configuration
2. Add all necessary transformers
3. Verify package relocation
4. Test shading with simple CXF service
5. Document shading process

**Week 3: Hybrid SOAP**
1. Implement WusHybridSoapVersionInterceptor
2. Test SOAP 1.1 envelope detection
3. Test WS-Addressing namespace acceptance
4. Create unit tests for interceptor
5. Integration test with actual SOAP messages

**Week 4: WS-Addressing**
1. Implement WusAddressingValidator
2. Test mandatory headers validation
3. Test OIN in URL support
4. Implement MessageID generation
5. Test RelatesTo correlation

**Success Criteria**:
- ✅ Shaded JAR contains no `org.apache.cxf` classes
- ✅ Can process SOAP 1.1 with WS-Addressing headers
- ✅ All Phase 1 unit tests pass
- ✅ Code coverage > 60%

### Phase 2: WS-Security Support (Weeks 5-7)

**Objectives**:
- Implement WS-Security for all three profiles
- PKIoverheid certificate handling
- Signature and encryption

**Deliverables**:
- ✅ WusSecurityConfigurator
- ✅ PKIoverheid certificate loading
- ✅ Profile implementations (BE, BE-S, BE-SE)
- ✅ Integration tests for each profile

**Tasks**:

**Week 5: Security Foundation**
1. Implement WusSecurityConfigurator
2. PKIoverheid crypto provider
3. Certificate chain validation
4. Test with self-signed certificates

**Week 6: Signing (Profile 2W-be-S)**
1. Configure WSS4J for signing
2. Implement signature parts selection
3. Test signature generation and validation
4. Implement SignatureConfirmation
5. Integration tests for signing

**Week 7: Encryption (Profile 2W-be-SE)**
1. Configure WSS4J for encryption
2. Implement encrypt-after-sign pattern
3. Test encryption/decryption
4. Integration tests for encrypted messages
5. Test combined signing + encryption

**Success Criteria**:
- ✅ All three profiles work (BE, BE-S, BE-SE)
- ✅ PKIoverheid certificates load correctly
- ✅ Signature validation passes
- ✅ Encryption/decryption works
- ✅ Profile-specific integration tests pass

### Phase 3: MTOM & Async Callbacks (Weeks 8-9)

**Objectives**:
- Add MTOM attachment support
- Implement async callback pattern
- Test all feature combinations

**Deliverables**:
- ✅ MTOM configuration support
- ✅ Async callback implementation
- ✅ Integration tests for MTOM and async

**Tasks**:

**Week 8: MTOM**
1. Configure MTOM in factories
2. Test attachment sending
3. Test attachment receiving
4. Test MTOM with WS-Security
5. Integration tests

**Week 9: Async Callbacks**
1. Implement callback endpoint hosting
2. Test ReplyTo with actual endpoint
3. Test MessageID/RelatesTo correlation
4. Integration tests for async pattern
5. Document async callback usage

**Success Criteria**:
- ✅ MTOM attachments work
- ✅ Async callbacks function correctly
- ✅ All combinations tested (MTOM + Security, Async + Security)

### Phase 4: Developer Experience (Weeks 10-11)

**Objectives**:
- Create easy-to-use factories
- Kotlin DSL wrapper
- Spring integration
- Documentation

**Deliverables**:
- ✅ WusJaxWsProxyFactoryBean (client)
- ✅ WusJaxWsServerFactoryBean (server)
- ✅ Kotlin DSL module
- ✅ Spring Boot auto-configuration
- ✅ Comprehensive examples

**Tasks**:

**Week 10: Java API**
1. Implement WusJaxWsProxyFactoryBean
2. Implement WusJaxWsServerFactoryBean
3. Fluent configuration API
4. Create examples (simple, secure, async)
5. Write usage documentation

**Week 11: Kotlin & Spring**
1. Create Kotlin module
2. Implement Kotlin DSL
3. Spring Boot starter module
4. Auto-configuration
5. Kotlin examples

**Success Criteria**:
- ✅ Factories are easy to use
- ✅ Kotlin DSL is idiomatic
- ✅ Spring integration works
- ✅ Examples run successfully

### Phase 5: Compliance & Hardening (Weeks 12-14)

**Objectives**:
- Pass Digikoppeling compliance tests
- Performance optimization
- Error handling improvement
- Security audit

**Deliverables**:
- ✅ Digikoppeling compliance certification
- ✅ Performance benchmarks
- ✅ Comprehensive error handling
- ✅ Security audit report

**Tasks**:

**Week 12: Compliance Testing**
1. Set up compliance test environment
2. Run all compliance tests
3. Fix any failures
4. Document compliance status
5. Generate compliance report

**Week 13: Performance & Hardening**
1. Performance profiling
2. Memory leak detection
3. Thread safety audit
4. Connection pool optimization
5. Error message improvement

**Week 14: Security Audit**
1. Security code review
2. Dependency vulnerability scan
3. Certificate handling audit
4. TLS configuration review
5. Fix any security issues

**Success Criteria**:
- ✅ 100% Digikoppeling compliance
- ✅ No critical security issues
- ✅ Performance benchmarks met
- ✅ All error paths tested

### Phase 6: Release Preparation (Weeks 15-16)

**Objectives**:
- Complete documentation
- Prepare for Maven Central (or internal repo)
- Final testing
- Release

**Deliverables**:
- ✅ Complete documentation
- ✅ Migration guide
- ✅ Release notes
- ✅ Published to Maven repository
- ✅ Release announcement

**Tasks**:

**Week 15: Documentation**
1. API documentation (JavaDoc)
2. User guide
3. Migration guide (from standard CXF)
4. Troubleshooting guide
5. FAQ

**Week 16: Release**
1. Final testing round
2. Version tagging (1.0.0)
3. Maven Central submission (or internal)
4. Release announcement
5. Community communication

**Success Criteria**:
- ✅ Documentation is complete
- ✅ Release artifacts published
- ✅ Release notes finalized
- ✅ Community informed

### Phase Timeline

```
Week  1 |█████████| Setup
Week  2 |█████████| Shading
Week  3 |█████████| Hybrid SOAP
Week  4 |█████████| WS-Addressing
Week  5 |█████████| Security Foundation
Week  6 |█████████| Signing
Week  7 |█████████| Encryption
Week  8 |█████████| MTOM
Week  9 |█████████| Async Callbacks
Week 10 |█████████| Java API
Week 11 |█████████| Kotlin & Spring
Week 12 |█████████| Compliance
Week 13 |█████████| Performance
Week 14 |█████████| Security Audit
Week 15 |█████████| Documentation
Week 16 |█████████| Release
```

**Total Duration**: 16 weeks (4 months)

---

## Team and Skills

### Team Structure

**Minimum Team**: 2 developers  
**Recommended Team**: 3 developers + 1 tester

### Role Definitions

#### Lead Developer

**Responsibilities**:
- Architecture decisions
- CXF internals expertise
- Code review
- Integration with CXF core

**Required Skills**:
- ✅ Deep Java knowledge
- ✅ SOAP/WSDL/WS-* standards
- ✅ Apache CXF internals
- ✅ Maven advanced (shading, plugins)
- ✅ Web service security

**Nice to Have**:
- Experience with framework customization
- Knowledge of Dutch government standards
- Previous SOAP security work

#### Developer #2

**Responsibilities**:
- WS-Security implementation
- PKIoverheid integration
- Testing
- Documentation

**Required Skills**:
- ✅ Java development
- ✅ XML Security
- ✅ TLS/PKI concepts
- ✅ JUnit/TestContainers
- ✅ SOAP basics

**Nice to Have**:
- WSS4J experience
- Cryptography knowledge
- Dutch government integration experience

#### Developer #3 (Optional but Recommended)

**Responsibilities**:
- Kotlin module
- Spring integration
- Examples
- Developer experience

**Required Skills**:
- ✅ Java & Kotlin
- ✅ Spring Boot
- ✅ API design
- ✅ Documentation

**Nice to Have**:
- DSL design experience
- Spring framework internals

#### QA/Tester (Optional but Recommended)

**Responsibilities**:
- Test strategy
- Compliance testing
- Performance testing
- Security testing

**Required Skills**:
- ✅ Integration testing
- ✅ Security testing
- ✅ Test automation
- ✅ Digikoppeling knowledge

### Skills Matrix

| Skill | Importance | Current Gap | Training Needed |
|-------|------------|-------------|-----------------|
| CXF Internals | 🔴 Critical | TBD | CXF architecture study, source code review |
| Maven Shade | 🟡 High | TBD | Plugin documentation, examples |
| SOAP Security | 🔴 Critical | TBD | WSS4J documentation, WS-Security specs |
| PKIoverheid | 🟡 High | TBD | PKIoverheid documentation |
| Digikoppeling | 🔴 Critical | TBD | Digikoppeling specifications |
| Kotlin | 🟢 Medium | TBD | Kotlin for Java developers |

### External Expertise

**Consider consulting with**:
- Apache CXF committers (for complex internals questions)
- Logius (for Digikoppeling clarifications)
- PKIoverheid experts (for certificate handling)

---

## Risk Assessment

### Technical Risks

| Risk | Probability | Impact | Mitigation Strategy |
|------|-------------|--------|---------------------|
| **Shading breaks CXF functionality** | Medium | High | Extensive testing at each phase, gradual shading approach |
| **Hybrid SOAP incompatibility** | Low | Critical | Early prototype, validate with Digikoppeling team |
| **WS-Security configuration issues** | Medium | High | Start with simple cases, incremental complexity |
| **Performance degradation** | Low | Medium | Performance benchmarks from day 1, profiling |
| **Certificate handling problems** | Medium | Medium | Use standard PKI libraries, thorough testing |
| **Compliance test failures** | Medium | High | Early compliance testing, iterative fixes |

### Project Risks

| Risk | Probability | Impact | Mitigation Strategy |
|------|-------------|--------|---------------------|
| **Team lacks CXF expertise** | High | High | Training, external consultant, documentation |
| **Scope creep** | Medium | Medium | Strict MVP focus, defer nice-to-haves |
| **Digikoppeling spec ambiguity** | Low | Medium | Direct communication with Logius |
| **Schedule overrun** | Medium | Medium | Buffer time (16 weeks vs 12 week target), prioritization |
| **Key person dependency** | Medium | High | Knowledge sharing, pair programming, documentation |

### Risk Response Plan

**For each critical risk**:

1. **CXF Functionality Breaks**
   - **Early Detection**: Automated tests after each shading change
   - **Response**: Rollback to last working state, investigate root cause
   - **Prevention**: Incremental shading, validate each step

2. **Compliance Test Failures**
   - **Early Detection**: Weekly compliance test runs
   - **Response**: Dedicated fix sprints, prioritize blockers
   - **Prevention**: Reference implementation study, early validation

3. **Team Expertise Gap**
   - **Early Detection**: Skills assessment in week 1
   - **Response**: Training program, pair with expert
   - **Prevention**: Hire experienced developer or consultant

---

## Deliverables

### Code Deliverables

1. **digikoppeling-wus-cxf-core-1.0.0.jar**
   - Shaded Apache CXF with WUS customizations
   - All three profiles supported (BE, BE-S, BE-SE)
   - MTOM and async callback support
   - Size: ~10-15 MB (shaded)

2. **digikoppeling-wus-cxf-kotlin-1.0.0.jar**
   - Kotlin DSL wrapper
   - Extension functions
   - Kotlin-friendly APIs
   - Size: ~200 KB

3. **digikoppeling-wus-cxf-spring-1.0.0.jar**
   - Spring Boot auto-configuration
   - Spring namespace handlers
   - Property-based configuration
   - Size: ~100 KB

4. **digikoppeling-wus-cxf-examples-1.0.0.jar**
   - Runnable examples
   - Client and server samples
   - All profile variations
   - Size: ~500 KB

### Documentation Deliverables

1. **User Guide** (50+ pages)
   - Getting started
   - Configuration reference
   - Profile selection guide
   - Common patterns
   - Troubleshooting

2. **API Documentation**
   - Complete JavaDoc for all public APIs
   - Kotlin DSL documentation
   - Code examples

3. **Migration Guide**
   - From standard CXF to WUS CXF
   - Configuration mapping
   - Code examples
   - Common pitfalls

4. **Developer Guide**
   - Architecture overview
   - Customization points
   - Contributing guidelines
   - Build and test instructions

### Test Deliverables

1. **Test Suite**
   - 200+ unit tests
   - 50+ integration tests
   - Compliance test results
   - Code coverage report (80%+)

2. **Test Certificates**
   - Test keystores
   - Test truststores
   - Certificate generation scripts

3. **Compliance Report**
   - Digikoppeling compliance results
   - Detailed test results
   - Known limitations

---

## Success Criteria

### Functional Success Criteria

✅ **Must Have (MVP)**:
1. Process SOAP 1.1 envelope with SOAP 1.2 WS-Addressing headers
2. Support all three WUS profiles (2W-be, 2W-be-S, 2W-be-SE)
3. Pass Digikoppeling compliance tests for all profiles
4. Handle PKIoverheid certificates correctly
5. Coexist with standard CXF in same application

✅ **Should Have**:
1. MTOM attachment support
2. Async callback pattern support
3. Kotlin DSL wrapper
4. Spring Boot integration
5. Comprehensive examples

✅ **Nice to Have**:
1. Performance optimizations
2. Advanced error recovery
3. Monitoring/metrics integration
4. WebFlux support (reactive)

### Technical Success Criteria

✅ **Code Quality**:
- Unit test coverage: 80%+
- Integration test coverage: All major scenarios
- No critical SonarQube issues
- No security vulnerabilities (OWASP, Snyk)

✅ **Performance**:
- Message processing overhead: < 5% vs standard CXF
- Memory footprint: < 50 MB for typical service
- Throughput: > 100 requests/second

✅ **Compatibility**:
- Java 11, 17, 21 support
- Kotlin 1.9+ support
- Spring Boot 2.7+ and 3.0+ support
- Maven and Gradle build support

### Project Success Criteria

✅ **Timeline**:
- MVP delivered in 12 weeks
- Full release in 16 weeks
- < 20% schedule variance

✅ **Quality**:
- Zero critical bugs at release
- < 5 known issues (documented)
- All compliance tests passing

✅ **Documentation**:
- Complete user guide
- All APIs documented
- Migration guide available
- 5+ working examples

✅ **Adoption**:
- Successfully used by 2+ pilot projects
- Positive feedback from early adopters
- No blocking issues reported

---

## Appendices

### Appendix A: Reference Documentation

**Digikoppeling**:
- [WUS Specification](https://logius-standaarden.github.io/Digikoppeling-Koppelvlakstandaard-WUS/)
- [Best Practices](https://logius-standaarden.github.io/Digikoppeling-Best-Practices-WUS/)
- [Compliance Suite](https://gitlab.com/logius/digikoppeling-compliance)

**Apache CXF**:
- [Official Documentation](https://cxf.apache.org/docs/)
- [WS-Security Guide](https://cxf.apache.org/docs/ws-security.html)
- [Interceptors](https://cxf.apache.org/docs/interceptors.html)

**Standards**:
- [WS-I Basic Profile 1.2](http://ws-i.org/Profiles/BasicProfile-1.2.html)
- [WS-I Basic Security Profile 1.1](http://ws-i.org/Profiles/BasicSecurityProfile-1.1.html)
- [WS-Addressing 1.0](https://www.w3.org/TR/ws-addr-core/)

### Appendix B: Development Environment Setup

**Required Tools**:
```bash
# Java
sudo apt install openjdk-17-jdk

# Maven
sudo apt install maven

# Git
sudo apt install git

# Docker (for testing)
sudo apt install docker.io docker-compose
```

**IDE Setup**:
- IntelliJ IDEA 2024.2+ (recommended)
- Eclipse 2024-06+ with Java 17 support
- VSCode with Java extensions

**Useful Plugins**:
- SonarLint (code quality)
- CheckStyle (style checking)
- Maven Helper (dependency analysis)

### Appendix C: Troubleshooting Common Issues

**Issue**: Shading breaks service discovery

**Solution**: Ensure `ServicesResourceTransformer` is configured

**Issue**: WS-Addressing headers not recognized

**Solution**: Check namespace in `WusHybridSoapVersionInterceptor`

**Issue**: Signature validation fails

**Solution**: Verify PKIoverheid truststore includes intermediate certificates

### Appendix D: Glossary

| Term | Definition |
|------|------------|
| **Digikoppeling** | Dutch government standard for service integration |
| **WUS** | Web Services profile of Digikoppeling (WSDL + UDDI + SOAP) |
| **Hybrid SOAP** | SOAP 1.1 envelope with SOAP 1.2 features |
| **Shading** | Package relocation to avoid classpath conflicts |
| **PKIoverheid** | Dutch government PKI infrastructure |
| **OIN** | Organisatie Identificatie Nummer (organization ID) |
| **2W-be** | Best Effort profile (TLS only) |
| **2W-be-S** | Signed profile (TLS + WS-Security signing) |
| **2W-be-SE** | Signed & Encrypted profile (TLS + signing + encryption) |

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-11-25 | Project Team | Initial version |

**Next Review Date**: Upon project completion

**Approval**:
- [ ] Technical Lead
- [ ] Project Manager
- [ ] Architecture Board

---

**End of Implementation Plan**
