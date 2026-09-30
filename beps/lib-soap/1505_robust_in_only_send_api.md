# Robust In-Only Send API for the SOAP Client

- Authors - @Nuvindu
- Reviewed by -
- Created date - 2026/09/29
- Updated date - 2026/09/29
- Issue - [#1505](https://github.com/ballerina-platform/ballerina-spec/issues/1505)
- State - Submitted

## Summary

Add a `send()` remote function to the SOAP 1.1 and SOAP 1.2 clients in the `ballerina/soap` module. The function implements the WSDL 2.0 Robust In-Only Message Exchange Pattern: it sends a SOAP envelope without expecting a response body, but checks the HTTP status code and returns SOAP fault information on failure.

## Motivation

The SOAP module currently provides two remote functions for sending messages:

- `sendReceive()` sends a SOAP envelope and returns the response body. It maps to the WSDL Request-Response Message Exchange Pattern.
- `sendOnly()` sends a SOAP envelope and ignores the response entirely, including any SOAP fault. It maps to the WSDL One-Way Message Exchange Pattern.

There is no way to send a one-way message while still detecting server-side failures. The WSDL 2.0 Robust In-Only MEP fills this gap: no response body is expected on success, but if the server returns a non-success HTTP status (such as 500 with a SOAP fault), the client reports the error. Users who call web services with one-way operations today must choose between losing fault information (`sendOnly`) or unnecessarily parsing a response body (`sendReceive`).

## Goals

- Send a SOAP envelope without expecting a response body on success.
- Detect non-2xx HTTP responses and return an error that includes the HTTP status code and any SOAP fault XML from the response.
- Support all existing client capabilities: custom HTTP headers, resource path suffixes, SOAP attachments via `mime:Entity[]`, and outbound security policies.
- Match the parameter style of `sendReceive()` and `sendOnly()` so users can switch between the three functions with minimal changes.

## Design

### 1. SOAP 1.1 Client

```ballerina
remote isolated function send(xml|mime:Entity[] body, string action,
                              map<string|string[]> headers = {}, string path = "") returns Error?
```

The `action` parameter is mandatory because SOAP 1.1 requires the `SOAPAction` HTTP header.

### 2. SOAP 1.2 Client

```ballerina
remote isolated function send(xml|mime:Entity[] body, string? action = (),
                              map<string|string[]> headers = {}, string path = "") returns Error?
```

The `action` parameter is optional because SOAP 1.2 carries the action in the `Content-Type` header and it is not always required.

### 3. Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `body` | `xml\|mime:Entity[]` | - | SOAP envelope as XML, or a SOAP message with MIME attachments |
| `action` | `string` (1.1) / `string?` (1.2) | - / `()` | SOAP action URI |
| `headers` | `map<string\|string[]>` | `{}` | Additional HTTP headers to include in the request |
| `path` | `string` | `""` | Resource path appended to the client endpoint URL |

### 4. Behavior

1. Outbound security policies configured on the client are applied to the SOAP envelope.
2. An HTTP POST request is created with the appropriate content type and action placement: `text/xml` with a separate `SOAPAction` HTTP header for SOAP 1.1, or `application/soap+xml` with the action as a `Content-Type` parameter for SOAP 1.2. Any custom headers are also included.
3. The request is sent to the server.
4. If the HTTP client returns an error (connection failure, timeout, etc.), an `Error` is returned with the client error message.
5. If the HTTP response status code is outside the 2xx range:
   - The function attempts to read the response body as XML (which may contain a SOAP fault).
   - An `Error` is returned with the HTTP status code and, if available, the SOAP fault XML in the error detail.
6. If the HTTP response status code is in the 2xx range, `()` is returned. The response body is not read.

### 5. Error detail

When a non-2xx response is received, the returned `Error` includes:

- **Message**: `"Failed to create SOAP response: HTTP <status code>"`
- **`httpStatusCode`**: The HTTP status code from the response.
- **`detail`**: The SOAP fault XML as a string, if the response body could be read as XML. Absent otherwise.

### 6. Comparison with existing functions

| Aspect | `sendReceive()` | `sendOnly()` | `send()` |
|--------|-----------------|--------------|----------|
| Expects response body | Yes | No | No |
| Checks HTTP status | Yes | No | Yes |
| Returns SOAP fault details | Yes (in response) | No | Yes (as error) |
| Return type | `xml\|mime:Entity[]\|Error` | `Error?` | `Error?` |
| WSDL MEP | Request-Response | One-Way | Robust In-Only |

### 7. Usage examples

#### SOAP 1.1: Send a one-way message with fault detection

```ballerina
import ballerina/soap.soap11;

public function main() returns error? {
    soap11:Client soapClient = check new ("http://www.example.com/soap-service");

    xml envelope = xml `<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"
                            soap:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
                            <soap:Body>
                              <quer:Add xmlns:quer="http://tempuri.org/">
                                <quer:intA>2</quer:intA>
                                <quer:intB>3</quer:intB>
                              </quer:Add>
                            </soap:Body>
                        </soap:Envelope>`;
    check soapClient->send(envelope, "http://tempuri.org/Add");
}
```

#### SOAP 1.2: Send a one-way message with fault detection

```ballerina
import ballerina/soap.soap12;

public function main() returns error? {
    soap12:Client soapClient = check new ("http://www.example.com/soap-service");

    xml envelope = xml `<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope"
                            soap:encodingStyle="http://www.w3.org/2003/05/soap-encoding">
                            <soap:Body>
                              <quer:Add xmlns:quer="http://tempuri.org/">
                                <quer:intA>2</quer:intA>
                                <quer:intB>3</quer:intB>
                              </quer:Add>
                            </soap:Body>
                        </soap:Envelope>`;
    check soapClient->send(envelope);
}
```

#### Handling a SOAP fault

```ballerina
import ballerina/soap.soap11;
import ballerina/io;

public function main() returns error? {
    soap11:Client soapClient = check new ("http://www.example.com/soap-service");

    xml envelope = xml `<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
                            <soap:Body>
                              <m:GetPrice xmlns:m="https://www.example.com/prices">
                                <m:Item>Apples</m:Item>
                              </m:GetPrice>
                            </soap:Body>
                        </soap:Envelope>`;
    soap11:Error? result = soapClient->send(envelope, "https://www.example.com/prices/GetPrice");
    if result is soap11:Error {
        io:println("SOAP call failed: ", result.message());
    }
}
```

## Alternatives

- **Use `sendReceive()` and discard the response.** This works but forces the client to read and parse a response body that is not expected, adding unnecessary overhead. It also misrepresents the intent of the operation.
- **Use `sendOnly()` and add error checking.** Not possible because `sendOnly()` does not inspect the HTTP response at all.
- **Add an optional parameter to `sendOnly()` to enable fault detection.** Rejected because it changes the semantics of an existing function and the two behaviors (ignore response vs. check status) are distinct enough to warrant separate functions.

## Testing

1. Successful `send()` call with a 2xx response, for both SOAP 1.1 and SOAP 1.2 clients.
2. `send()` with a non-2xx response containing a SOAP fault, verifying the error includes the HTTP status code and the fault XML.
3. `send()` with a connection error (unreachable host), verifying the error is propagated.
4. `send()` with custom headers and a non-empty path.
5. `send()` with `mime:Entity[]` body (SOAP attachments).
6. `send()` with outbound security policies applied to the envelope.

## Risks and Assumptions

- The function does not read the response body on success. If a server returns meaningful data in a 2xx response to a one-way operation, that data is silently discarded. Users who need the response body should use `sendReceive()`.
- SOAP fault extraction from the error response is best-effort. If the response body is not valid XML, the error is still returned but without the fault detail.

## Dependencies

None.

## Future Work

- Inbound security policy support if a use case for processing signed or encrypted fault responses arises.

## References

- [WSDL 2.0 Robust In-Only MEP](https://www.w3.org/TR/wsdl20-adjuncts/#in-only)
