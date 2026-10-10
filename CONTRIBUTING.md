# Contributing

Thanks for your interest in this project. Contributions in the form of bug reports, fixes, and documentation improvements are welcome.

## Reporting a problem

The two areas still being tuned on real hardware are **audio (ALC285 layout)** and the **trackpad (ELAN1010)**. If you can help, please open an issue with as much of the following as possible:

### Required information

1. **Hardware** — confirm it is an ASUS ROG Zephyrus M16 GU603ZM (or state your exact model / BIOS version).
2. **What you changed** — which config / kext / layout you tried.
3. **What you observed** — the exact symptom (photo or text log is best).

### Getting diagnostics (recommended)

Boot into macOS and run this one-liner (it writes a full report to your USB stick):

```bash
bash /Volumes/<USB>/g.sh
```

The report covers: model/CPU name, GPU driver state, audio device tree, trackpad (`VoodooI2CPCIController` load state), GPIO, network/Bluetooth, and NVRM kernel logs. Attach the `报告-*.txt` it produces to your issue.

For GPU/driver issues specifically, the key log line is:

```bash
log show --last 1d --predicate 'eventMessage CONTAINS "NVRM-xnu"' | tail -50
```

And the driver's decision fields:

```bash
ioreg -r -n NVRM -d 1 -l | grep nvrm-
```

### Audio layout rotation

If you found a working `layout-id` for ALC285 on this machine, please report it — the current recommendation is still unconfirmed.

## Pull requests

- Keep changes scoped to one logical fix.
- Run `ocvalidate` against `EFI/OC/config.plist` (must report "No issues found").
- Never commit real SMBIOS triplets (Serial/MLB/UUID/ROM) — keep them as placeholders.

## License

By contributing, you agree that your contributions will be licensed under the [MIT License](LICENSE).
