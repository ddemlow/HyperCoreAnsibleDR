# Contributing

Contributions are welcome. Please open an issue before submitting a pull request for
significant changes, so the approach can be discussed first.

## Development setup

```
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
```

## Test environment

Testing requires:

- Two HyperCore clusters running ICOS v9.4 or later (source and DR)
- Replication configured from source to DR
- An Ansible controller that can reach both cluster APIs and ping all node IPs

Run the playbook in check mode against your DR cluster to verify syntax:

```
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml --check
```

Note: check mode will not fully validate HyperCore API calls since the HyperCore modules
do not all support check mode.

## Submitting changes

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/my-change`)
3. Commit your changes
4. Open a pull request against `main`

Please keep pull requests focused on a single change. Update `README.md` if any variables,
behavior, or requirements change.

## Reporting issues

Use [GitHub Issues](../../issues) to report bugs or request features. For bugs, include:

- HyperCore ICOS version
- `scale_computing.hypercore` collection version (`ansible-galaxy collection list scale_computing.hypercore`)
- Ansible version (`ansible --version`)
- Playbook output (sanitize any passwords or sensitive IP addresses)
