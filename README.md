# Zebra ZD621 Universal Orchestrator Extension

Keyfactor Universal Orchestrator extension for the HTTPS server certificate on Zebra ZD621 Link-OS printers. It targets Keyfactor Command 12+ and implements:

- Discovery of ZD621 printers from host lists or IPv4 CIDR ranges
- Inventory of the active HTTPS certificate
- Add and replace from a Keyfactor-delivered PKCS#12/PFX
- Removal of the HTTPS certificate files

The extension uses Zebra SGD/ZPL over TCP raw port 9100. It does not require the Zebra Java SDK. Inventory performs a TLS handshake against the configured HTTPS port and records the certificate actually in use.

## Important security note

TCP raw port 9100 is unencrypted and unauthenticated on standard Link-OS configurations. Restrict it with network ACLs so only the Universal Orchestrator host can reach it. Use a segregated provisioning network when possible. After provisioning, Zebra recommends disabling unencrypted services; doing that requires an alternate secured management path for future renewals.

## Build

Requirements: .NET 8 SDK and network access to NuGet.

```powershell
dotnet restore .\ZebraPrinterOrchestrator.sln
dotnet test .\ZebraPrinterOrchestrator.sln -c Release
dotnet publish .\src\ZebraPrinterOrchestrator\ZebraPrinterOrchestrator.csproj -c Release -o .\publish
Compress-Archive -Path .\publish\* -DestinationPath .\ZebraPrinterOrchestrator.zip -Force
```

## Install

1. Run `scripts/create-store-type.ps1` with a Keyfactor API bearer token.
2. Copy the published files into an extension folder under the Universal Orchestrator installation, for example `extensions/ZebraZD621`.
3. Restart the Universal Orchestrator service.
4. Create a store with:
   - Client Machine: printer DNS name or IP
   - Store Path: `zebra://printer-host:9100/HTTPS`
   - No store credential is required by the printer command channel

The extension capability/short name is exactly `ZebraZD621` and must match the store type.

## Discovery

Enter comma/semicolon-separated hosts or an IPv4 CIDR in the discovery job's **Directories to search** field. Examples:

```text
printer01,printer02,10.20.30.41
10.20.30.0/24
```

Ranges are limited to 4,096 addresses and are probed with up to 32 concurrent connections. A target is returned only when `device.product_name` contains `ZD621`.

## Certificate behavior

On add/replace, the extension:

1. Opens the PFX delivered by Keyfactor.
2. Writes the unencrypted private key followed by the leaf and chain as PEM.
3. Deletes old `HTTPS_CERT.NRD`, `HTTPS_KEY.NRD`, and `HTTPS_CA.NRD` objects.
4. Uploads a single `E:HTTPS_CERT.NRD` with Zebra's `~DY` binary command.
5. Confirms that the object exists and requests a printer reset.

Zebra documents that `HTTPS_KEY.NRD` cannot contain an encrypted private key. The unencrypted key exists only in memory in this implementation and is never logged, but the port-9100 transport itself is not encrypted.

On Link-OS 7.4.2 and newer, removing the user certificate can cause the printer to generate a self-signed certificate. Inventory intentionally reports a certificate only when `HTTPS_CERT.NRD` exists, so the generated fallback is not treated as a managed entry.

## Compatibility and validation

This repository targets the current Keyfactor extension interface package (`Keyfactor.Orchestrators.IOrchestratorJobExtensions` 0.7.0) and .NET 8. The code includes unit tests for endpoint parsing and CIDR expansion. A live-printer validation is still required because Zebra firmware can change command-channel availability; verify at least:

- `! U1 getvar "device.product_name"`
- `! U1 do "file.dir" "E:"`
- `~DYE:HTTPS_CERT.NRD,B,NRD,<size>,,<content>`
- `! U1 do "device.reset" ""`

Test first on a non-production ZD621 running the same firmware as production.
