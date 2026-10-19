# What an iOS App Leaks When You Just Unzip It

**Subtitle:** Two DeadFace CTF mobile challenges solved without an emulator, a debugger, or a jailbreak — only `unzip` and a bit of reading.

A lot of people assume mobile analysis needs a jailbroken phone and a pile of dynamic-analysis tooling. For these two challenges you needed none of it. An `.ipa` is just a ZIP archive, and almost everything that mattered fell straight out of the bundle. I'm covering both "Aurora Patient Portal" and its follow-up "Back to Basics" here, because they're the same app and the second builds directly on the first.

## The app

Both challenges used `AuroraPatientPortal.ipa`, a healthcare app that was leaking patient data DeadFace used for phishing. The hint said plainly that no emulator or debugger was needed — everything was static inside the bundle. Unzipping it gave the usual iOS layout:

```
Payload/aurora-patient-portal.app/
    Info.plist
    aurora_meds.sqlite
    <a chest X-ray JPG>
    Aurora Health Mobile      (the Mach-O binary)
```

## Challenge 1 — Aurora Patient Portal

**Objective:** find the name and date of birth of the last patient who used this build. Flag format: `deadface{firstname-lastname-YYYY-MM-DD}`.

The bundled database, `aurora_meds.sqlite`, held 100 patients keyed by `pid` but carried no names or birthdates, so the "last user" lived elsewhere. The X-ray's XMP metadata pointed at `dc:identifier = PID-847261` (a real `pid` in the database, but still no name). The decisive artifact was a key in `Info.plist`:

```python
import zipfile, plistlib
z = zipfile.ZipFile('AuroraPatientPortal.ipa')
n = [x for x in z.namelist() if x.endswith('.app/Info.plist')][0]
print(plistlib.loads(z.read(n)).get('AuroraPatientBootstrap'))
# eyJwYXRpZW50TmFtZSI6Ik1hcmEgVm9zcyIsImRhdGVPZkJpcnRoIjoi
# MTk4OC0wNC0xNyIsImNsaW5pYyI6Ildlc3RsYWtlIn0=
```

That value is base64. Decoding it:

```json
{"patientName":"Mara Voss","dateOfBirth":"1988-04-17","clinic":"Westlake"}
```

Reformatted to the requested pattern:

```
deadface{Mara-Voss-1988-04-17}
```

The developer had "bootstrapped" the last patient's details into `Info.plist` as a base64 blob, apparently treating base64 as a form of protection. It isn't — it's reversible encoding that anyone who unzips the app can read.

## Challenge 2 — Back to Basics

**Objective:** name the gateway server the app uses to reach Aurora's backend. Flag format: `deadface{gateway-name}`.

Symbol names in the Mach-O binary showed a SwiftUI `DiagnosticScreen` with rows like "Last Gateway Check," which told me the gateway host was almost certainly a string compiled into the binary. So I read printable strings out of the binary and filtered for gateway-like tokens:

```python
import zipfile, re
z = zipfile.ZipFile('AuroraPatientPortal.ipa')
n = [x for x in z.namelist() if x.endswith('.app/Aurora Health Mobile')][0]
d = z.read(n)
for s in re.findall(rb'[ -~]{4,}', d):
    if any(k in s.lower() for k in [b'gw-', b'gateway', b'-int-']):
        print(s.decode())
# gw-westlake-int-03
# Last Gateway Check
# Internal clinic sync status
```

Only one gateway token existed, and `westlake` matched the clinic from the first challenge's bootstrap blob — which is what "use everything you collected" was pointing at.

```
deadface{gw-westlake-int-03}
```

## Why it works

Neither challenge needed runtime analysis because the sensitive data was baked into shipped resources. The patient details sat in `Info.plist` behind base64; the gateway hostname sat as a plain string in the binary. Both are realistic mobile findings — developers routinely ship secrets, endpoints, and PII inside the app package assuming nobody will look. Unzipping the package is "looking."

## What I took from it

Treat `.ipa` and `.apk` files as archives first; most static secrets fall out of the bundle before you need anything fancier. `Info.plist`, bundled SQLite databases, and image EXIF/XMP are the first places to check. Base64 is encoding, not encryption, so finding PII stored that way is a finding, not a dead end. And symbol names in a Swift binary (like `DiagnosticScreen`) tell you what features exist even without full reversing — enough, here, to know a hardcoded hostname was waiting in the strings.

---

**Medium excerpt:** No emulator, no jailbreak — just `unzip`. Here's how two DeadFace CTF mobile challenges gave up patient PII and a backend hostname straight from the iOS app bundle.

**Tags:** Cybersecurity · Capture The Flag · Mobile Security · iOS · Infosec

**Cover image concept:** An `.ipa` file icon "unzipping" into its contents (Info.plist, a .sqlite, an X-ray, a binary) with a base64 blob decoding to a name tag. Sells the "it's just a zip" message.
