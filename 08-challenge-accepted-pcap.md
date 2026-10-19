# Catching an NTLMv2 Hash from a Poisoning Attack

**Subtitle:** Following an LLMNR poisoning attack through a packet capture to the exact byte an attacker wants: the NTProofStr.

Network forensics clicked for me on this challenge, because it isn't abstract — you're watching a real, common attack unfold packet by packet. "Challenge Accepted" was a 100-point **Network Forensics** challenge built around an LLMNR/NBT-NS poisoning capture, and the goal was to find the victim's first NTLMv2 authentication after the poison and read its 16-byte NTProofStr.

## The scenario

This is the textbook Responder setup. A victim tries to resolve a hostname, name resolution falls through to LLMNR broadcast, and the attacker answers first claiming to be that host. The victim then authenticates to the attacker over SMB and leaks an NTLMv2 response. Parsing the capture (`demonne.pcap`) with scapy and a small NTLMSSP parser established the cast:

- Victim ("Nathan"): user `nkessler`, workstation `DMF-WS11-042`, `10.77.10.20`
- Attacker (Responder/Kali): `10.77.10.66`
- Poisoned name: `dmf-field-01`

## Finding the poison, then the auth

The poisoning event is the first LLMNR response from the attacker claiming the queried name:

```
pkt #1463   10.77.10.66 -> 10.77.10.20
            LLMNR response  name=dmf-field-01  answer=10.77.10.66
```

The very next NTLMSSP_AUTH (Type 3) message from the victim to the attacker is Nathan's first authentication after being poisoned:

```
pkt #1489   10.77.10.20 -> 10.77.10.66 :445
            USER=nkessler  HOST=DMF-WS11-042  (NTLMv2, ntlen=332)
```

In Wireshark the same packet is trivial to find with a display filter:

```
ntlmssp.messagetype == 3
```

Frame 1489 was the first hit. Expanding **NTLM Secure Service Provider → NTLM Response → NTLMv2 Response** shows the field directly:

```
NTProofStr: 5a2342014fe7f28997ce4dcd09f6b7f8
```

```
deadface{5a2342014fe7f28997ce4dcd09f6b7f8}
```

## Why this matters

When name resolution falls back to LLMNR or NBT-NS broadcast, any machine on the segment can answer. Responder just answers first, claiming the requested name, so the victim connects to the attacker and performs NTLMv2 authentication against it. The NTProofStr is the first 16 bytes of the `NtChallengeResponse` — an HMAC-MD5 that, together with the server challenge and the rest of the blob, can be fed to hashcat (mode 5600) and cracked offline into the user's password. Capturing that value is the entire point of the attack, which is why the challenge asks for it specifically.

## What I took from it

Two display filters do most of the work in a capture like this: one to find the poisoned name response, and `ntlmssp.messagetype == 3` to isolate the authentication messages that actually carry the proof. It helps to know the NTLMv2 response layout well enough to point at NTProofStr as its first 16 bytes rather than hunting around. And the defensive takeaway is simple and worth saying out loud in a portfolio: disabling LLMNR and NBT-NS removes this whole class of credential theft.

---

**Medium excerpt:** An LLMNR poisoning attack, captured and followed packet by packet to the one value the attacker is really after. Here's how I pulled the NTProofStr from a DeadFace CTF network-forensics challenge.

**Tags:** Cybersecurity · Capture The Flag · Network Forensics · Wireshark · Infosec

**Cover image concept:** A simple three-node diagram — victim, attacker, and the poisoned LLMNR reply — with an NTLMv2 packet flowing to the attacker and the NTProofStr bytes highlighted. Or a cropped Wireshark pane showing the `ntlmssp.messagetype == 3` filter and the NTProofStr field.
