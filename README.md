**Update `README.md`**:
```bash
cat > README.md <<EOF
# SailPoint IdentityIQ Test Lab

This repository documents a test lab for SailPoint IdentityIQ (IIQ) with FreeIPA for LDAP-based identity management. It includes scripts and documentation for a non-production environment. **No proprietary SailPoint code or sensitive data is included.**

## Structure
- **docs/**: Setup guides and troubleshooting.
- **scripts/**: Safe scripts for FreeIPA setup and user provisioning.
- **configs/**: Generic configuration templates (e.g., LDIF files).

## Getting Started
1. Clone the repository:
\`\`\`bash
git clone https://github.com/kennth-nnadi/sailpoint-iiq-test-lab.git
\`\`\`
2. Follow guides in \`docs/\` to set up FreeIPA and configure IdentityIQ.
3. Use scripts in \`scripts/\` for automation.

## Legal Note
This repository is for educational purposes and contains no proprietary SailPoint code or sensitive data, in compliance with legal restrictions.

## License
MIT License
EOF
