# bcrypt Security Lab

A hands-on lab exploring how bcrypt password hashing works, how resistant it is to offline attacks, and a database-tampering vulnerability discovered through independent testing — all built and run on a local, self-owned environment.

> ⚠️ **Disclaimer:** All testing in this repository was performed exclusively against a local test application and database that I built and own. No real user data, live systems, or third-party credentials were used at any point. This project is for educational purposes only. See [DISCLAIMER.md](./DISCLAIMER.md) for the full statement.

---

## What this project covers

- **bcrypt internals** — salting, cost factors, the Eksblowfish cipher, and how the final hash string is structured
- **Cost factor benchmarking** — measuring hashing speed and crack time across different work factors
- **Offline dictionary attacks** — simulating how an attacker with a stolen hash (but no live server access) can attempt unlimited guesses
- **A hash-swapping vulnerability** — a database-tampering technique discovered while testing, which bypasses password verification entirely regardless of hash strength

## Key finding

Swapping two users' bcrypt hashes directly in the database allowed each user to log in using the *other* user's password. This confirms that bcrypt protects against offline hash cracking, but provides no protection against an attacker (or insider) with direct write access to the database. Full write-up: [`docs/hash-swap-writeup.md`](./docs/hash-swap-writeup.md).

## Repository structure

```
bcrypt-security-lab/
├── README.md
├── DISCLAIMER.md
├── LICENSE
├── requirements.txt
├── docs/
│   ├── how-bcrypt-works.md
│   ├── attack-vectors.md
│   └── hash-swap-writeup.md
├── src/
│   ├── offline_attack.py
│   ├── benchmark_cost_factors.py
│   └── init_db.py
└── results/
    └── benchmark_data.csv
```

## How to run it

**Requirements:** Python 3.10+, pip

```bash
git clone https://github.com/abhinav-kamoji/bcrypt-security-lab.git
cd bcrypt-security-lab
pip install -r requirements.txt
```

**Set up the local test database** (creates a fresh local environment — no real data):

```bash
python src/init_db.py
```

**Run the cost factor benchmark:**

```bash
python src/benchmark_cost_factors.py
```

**Run the offline dictionary attack simulation** (against a hash generated in this local environment):

```bash
python src/offline_attack.py
```

## Results summary

Benchmarked hashing time and dictionary attack performance across bcrypt cost factors 8–12 on local hardware:

| Cost Factor | Rounds | Avg. Hash Time | Notes |
|---|---|---|---|
| 8   | 256   | ~15ms  | Fast, weak by 2026 standards |
| 10  | 1,024 | ~65ms  | Old default, no longer sufficient alone |
| 12  | 4,096 | ~250ms | Current recommended minimum |
| 13  | 8,192 | ~500ms | Recommended if server latency allows |

Full data in [`results/benchmark_data.csv`](./results/benchmark_data.csv).

Weak, dictionary-based passwords (`password`, `123456`, `qwerty`) were cracked in under a second regardless of cost factor. Strong, random 12+ character passwords remained uncracked against the same wordlist — illustrating that password *strength* matters more than hashing speed once you're above a sane minimum cost factor.

## What I learned

- bcrypt is designed to be slow on purpose, and that slowness is the actual security mechanism against offline cracking — not secrecy of the salt, which is stored in plain sight.
- Rate limiting on a login form stops *online* attacks but does nothing against an attacker who has already stolen the password hash.
- Password hashing and access control are separate layers of defense. A cryptographically sound hash does not protect against an application or database that doesn't enforce who can write to it.
- Defense in depth (access control, input validation, 2FA, audit logging) is necessary specifically because no single layer — including a strong hash — covers every failure mode.

## What I'd do next

- Add an Argon2id implementation and benchmark it head-to-head against bcrypt
- Add rate limiting and account lockout to the test app and measure their actual effect
- Explore mitigations for the hash-swap vulnerability at the application layer (e.g., binding the hash to a user identifier)

## License

MIT — see [LICENSE](./LICENSE).
