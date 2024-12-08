# Dynamic DRM Challenge Handling for PlayReady API

This repository documents the process of integrating a DRM-related PlayReady system into a custom application, `VineTrimmer`. The integration focuses on dynamically handling DRM challenges, parsing `pssh` objects, and resolving playback issues that arise due to mismatched API responses.

---

## Agenda 1: Enhancements to PSSH Parsing

Previously, libraries like `pyplayready` did not support parsing `pssh` objects. To address this, the following custom `PSSH` class was added to handle parsing, validating, and extracting essential components such as `WRMHeader` XML, license URLs, and Key IDs.

### Code Example: `PSSH` Class
```python
class PSSH(_PlayreadyPSSHStructs):
    SYSTEM_ID = UUID(hex="9a04f07998404286ab92e65be0885f95")

    def __init__(self, pssh_data: Union[str, bytes]):
        if isinstance(pssh_data, str):
            try:
                pssh_data = base64.b64decode(pssh_data)
            except Exception as e:
                raise InvalidPssh(f"Failed to decode Base64 PSSH: {e}")

        try:
            parsed = self.PSSHBox.parse(pssh_data)
            system_id = UUID(bytes=parsed.system_id)
            if system_id != self.SYSTEM_ID:
                raise InvalidPssh(f"Invalid System ID: {system_id}")

            self.raw_data = parsed.data
            self.wrmheader = self._extract_wrmheader(parsed.data)

        except ConstructError as e:
            raise InvalidPssh(f"Failed to parse PSSH Box: {e}")

    def _extract_wrmheader(self, data: bytes) -> Element:
        try:
            xml_string = data.decode("utf-16-le")
            return fromstring(xml_string)
        except Exception as e:
            raise InvalidPssh(f"Failed to parse WRMHeader XML: {e}")

    def get_wrmheader_xml(self) -> str:
        return tostring(self.wrmheader, encoding="unicode")

    def get_license_url(self) -> str:
        try:
            la_url = self.wrmheader.find(".//{http://schemas.microsoft.com/DRM/2007/03/PlayReadyHeader}LA_URL")
            return la_url.text if la_url is not None else None
        except Exception as e:
            raise InvalidPssh(f"Failed to extract License URL: {e}")

    def get_key_id(self) -> str:
        try:
            kid = self.wrmheader.find(".//{http://schemas.microsoft.com/DRM/2007/03/PlayReadyHeader}KID")
            return kid.attrib.get("VALUE") if kid is not None else None
        except Exception as e:
            raise InvalidPssh(f"Failed to extract Key ID: {e}")

    @staticmethod
    def _read_playready_objects(header: Container) -> List[WRMHeader]:
        return list(map(
            lambda pro: WRMHeader(pro.data),
            filter(
                lambda pro: pro.type == 1,
                header.records
            )
        ))
```
### Parsed PSSH Example:
- The screenshot below highlights the successfully parsed pssh object, including extracted WRMHeader, License URL, and Key ID.

![vtpr-debug1](https://github.com/user-attachments/assets/52d68319-3b86-4777-8d43-87d96643414d)

# Agenda 2: Playback Request Issue
- During playback, a significant issue occurs when the API returns the following error:
```
HTTP Request: POST /ng/msl_v1/cadmium/pbo_licenses...
This title is not available to watch instantly. Please try another title.
InvalidChallenge (nq_cadmium_pbo_License82.157.19/pbo-client82.157.0)
```
### Issue Breakdown
Problem 1: Challenge Parameter Handling
- Configuration: The code uses self.config["payload_challenge"] to set the challenge parameter.
- API Behavior: The API returns a dynamic challenge value in the first response. However, the code uses the static value from self.config["payload_challenge"], leading to invalid challenge submissions.
Problem 2: Data Flow Mismatch
- Challenge Update: The current mechanism does not dynamically update the payload_challenge parameter with the value returned by the API.
- Impact: Without properly updating the challenge, subsequent license requests fail, causing playback errors.
Problem 3: API Requirements
- PlayReady APIs require the exact challenge returned by the API to be reused in subsequent requests. Using static or outdated values leads to mismatches and errors.
