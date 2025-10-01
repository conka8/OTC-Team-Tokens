# MetaHash SN73 — Team OTC Sell Guide (Miner Onboarding → TAO)

**Goal:** Start with **Team Tokens (α)** in your team wallet (e.g., SN20). End with **TAO in your team wallet** by selling α via **Subnet 73 (MetaHash)**.

> Plain words. Tiny steps. Copy/paste where possible.

---

## TL;DR (Super Simple)

1. **Make wallet.** Coldkey (keeps TAO). Hotkey (does work).
2. **Put a little TAO** in **coldkey** (fees to register & run).
3. **Register hotkey on SN73.** (So you can bid.)
4. **Hold α** (team tokens) in **source hotkeys** you control.
5. **Run the SN73 miner.** It sends **bids** like: `(subnet_id, α_amount, discount_bps)`.
6. **Wait for `Win`.** It shows a **pay window** `[as, de]` (block range).
7. **Send α** to the **treasury** **within the window**.
8. **Validator checks & settles.** You get **weights → emissions**.
9. Emissions become **staked TAO on your hotkey**.
10. **Unstake/Recycle** → **TAO liquid** on your **coldkey**.

**You traded α → TAO** using the SN73 auction. That’s OTC-like, but automated and safe.

---

## Picture (Flow)

```
Team α  ──► SN73 Miner (your hotkey)
           │   send bids
           ▼
      SN73 Validator Auction ──► Win + Pay Window [as,de]
           │                          ▲
           │ verify α paid to treasury │
           ▼                          │
      Settlement → Set Weights → Emissions (TAO to your hotkey stake)
           ▼
   Unstake / Recycle → TAO liquid on your coldkey
```

---

## What You Need

* **Machine**: Linux VPS is fine.
* **Python 3.10+** and `venv`.
* **MetaHash code**: `https://github.com/fx-integral/metahash`
* **btcli** (Bittensor CLI) on your box.
* **TAO**: a tiny amount on **coldkey** for fees.
* **α**: your Team Tokens on **source hotkeys** (these will fund the payments when you win bids).

> Note: Validators only accept α sent to **whitelisted treasuries**. The list lives in `metahash/treasuries.py`.

---

## Quick Install (Miner)

```bash
# Get code
git clone https://github.com/fx-integral/metahash.git
cd metahash

# Python env
python -m venv .venv && source .venv/bin/activate
pip install -U pip wheel uv
uv pip install -e .

# Env file
cp .env.template .env
# Edit .env → set WALLET_PASSWORD (and BITTENSOR_NETWORK if needed)
```

---

## Make Wallet & Hotkey; Register on SN73

```bash
# Create coldkey (stores TAO)
btcli wallet new_coldkey --wallet.name team

# Create hotkey (does work)
btcli wallet new_hotkey --wallet.name team --wallet.hotkey miner1

# Fund coldkey with a little TAO for fees (from exchange or another wallet)

# Register hotkey on SN73 (pick the command your btcli supports)
# Option A (newer):
btcli subnets register --netuid 73 --wallet.name team --wallet.hotkey miner1
# Option B (older):
btcli register --netuid 73 --wallet.name team --wallet.hotkey miner1
```

**Where do fees/TAO sit?**

* **Registration fees** come from **coldkey**.
* **Emissions** accrue as **staked TAO on your hotkey**. Later you **unstake/recycle** to coldkey to make TAO liquid.

---

## Prep Your α (Team Tokens)

* Put **α** (e.g., SN20 stake) into the **source hotkeys** that will fund payments.
* You will point the miner to these with `--payment.validators <hotkeyA> <hotkeyB> ...`.
* When you **win**, α is **sent from these hotkeys** to the **treasury** addresses.

> If `STRICT_PER_SUBNET=true`, each accepted bid line must be paid on **that specific subnet**.

---

## Bids = (subnet_id, α_amount, discount_bps)

* `subnet_id`: where you supply α (ex: 20, 71, 72, 73…)
* `α_amount`: how much α to sell on that subnet
* `discount_bps`: **basis points** (1 bp = 0.01%). Example: `500 = 5%`, `700 = 7%`, `2500 = 25%`.

**Why discount?**

* Each subnet has a weight (ex: `0.8` → base 20% haircut).
* Your `discount_bps` is the **max extra haircut** you accept **including** that base.
* If base haircut > your max discount → **bid won’t be sent** (safety).

**Examples**

* Base 0.8 (20%): Your max 25% (2500 bps) → effective 5% extra → You get 75% value.
* Base 0.95 (5%): Your max 25% → effective 20% → You get 75% value.
* Base 0.7 (30%): Your max 25% → too high → bid not sent.

**Modes**

* Default: effective-discount (scaled by subnet weight).
* Raw: add `--miner.bids.raw_discount` to use your bps as-is.

---

## Run the Miner (Example)

```bash
python neurons/miner.py \
  --netuid 73 \
  --wallet.name team \
  --wallet.hotkey miner1 \
  --subtensor.network "ws://YOUR_NODE:9944" \
  --miner.bids.netuids 20 71 73 \
  --miner.bids.amounts 100.0 50.0 25.0 \
  --miner.bids.discounts 700 1200 2500 \
  --axon.port 8091 \
  --axon.external_port 8091 \
  --logging.debug \
  --payment.validators HOTKEY_A HOTKEY_B
```

**Notes**

* `--payment.validators` = the **hotkeys** that hold your α. Payments are drawn from them.
* You can use `pm2` to keep it alive.

---

## What Happens Each Epoch (e → e+1 → e+2)

1. **e: Auction** starts; miner sends bids.
2. **e: Clearing**; if you win → you get a **Win invoice** with a **pay window** `[as, de]` (block range in **e+1**).
3. **e+1: Pay** α **within [as,de]** to the correct **treasury** (the miner and validator have this list). If `STRICT_PER_SUBNET=true`, pay per line per subnet.
4. **e+2: Settlement**; validator checks payments, burns underfill to UID 0, **sets weights**.
5. Emissions → **TAO** stake on your **hotkey**.
6. **Unstake/Recycle** → TAO liquid on your **coldkey**.

> Missed window or wrong treasury/subnet? That line is ignored. Underfill is **burned** (to UID 0). No weight for that part.

---

## From Emissions to Liquid TAO (Cash Out)

1. Check your hotkey stake growth (emissions):

   ```bash
   btcli wallet overview --wallet.name team
   ```
2. Move TAO from **hotkey stake** to **coldkey liquid** (command names vary by btcli version; examples):

   ```bash
   # Example: remove stake (older)
   btcli stake remove --wallet.name team --wallet.hotkey miner1 --amount 10

   # Or: recycle / unstake to coldkey (some builds)
   btcli wallet recycle --wallet.name team --wallet.hotkey miner1 --amount 10
   ```
3. Now your **coldkey** has liquid TAO. Send it where you need (exchange, treasury coldkey, etc.).

> Fees apply. Use small test amounts first.

---

## OTC, but Safer (Why This Works for Teams)

* You **don’t** DM strangers. You use the **auction**.
* Your **max discount** is enforced in code.
* α only goes to **whitelisted treasuries**.
* If you or the chain are slow (e.g., only **few T per block**), the **window spans many blocks**, so you can still fill.
* If you fail to fill, the missing part is **burned** (not misallocated), so totals stay honest.

**Daily caps?** Budgets/limits are **config** (e.g., `AUCTION_BUDGET_ALPHA`). They can change. Check the validator’s config/announcements.

---

## Team Tutorial: Rizzo has SN20 α → Ends with TAO (Step-by-Step)

**We assume:** Rizzo controls `team` (coldkey) and `miner1` (hotkey). Rizzo has α on hotkeys `HOTKEY_A, HOTKEY_B`.

### 0) Prep

* Install MetaHash, make wallet/hotkey, register on SN73 (see above).
* Put a **little TAO** on **coldkey** for fees.
* Ensure **α** is on **HOTKEY_A/HOTKEY_B**.

### 1) Start Miner & Send Bids

```bash
python neurons/miner.py \
  --netuid 73 \
  --wallet.name team \
  --wallet.hotkey miner1 \
  --subtensor.network "ws://YOUR_NODE:9944" \
  --miner.bids.netuids 20 \
  --miner.bids.amounts 100.0 \
  --miner.bids.discounts 900 \
  --payment.validators HOTKEY_A HOTKEY_B \
  --logging.debug
```

* Here, Rizzo is willing to sell **100 α on SN20** with **max 9% discount**.

### 2) Watch Logs → `Win`

* You will see `Win` with a **pay window** `[as, de]` that lands in **epoch e+1**.
* The log also shows the **treasury** address for the line.

### 3) Pay α **within [as,de]**

* If configured, your miner will send α from `HOTKEY_A/B` to the treasury during the window.
* If you do it **manually**, double-check:

  * **Subnet** is correct (SN20 line pays on SN20).
  * **Treasury address** matches `metahash/treasuries.py`.
  * Amounts match your **accepted** line(s).

### 4) Settlement (e+2)

* Validator scans chain, merges windows, applies `STRICT_PER_SUBNET` if set.
* Any **underfill** gets **burned**.
* Validator **sets weights** for your hotkey.

### 5) Emissions → Unstake → TAO on Coldkey

* Emissions appear as **staked TAO on `miner1`**.
* Use **unstake/recycle** to move to **coldkey**.

**Done:** Rizzo started with **SN20 α** and ended with **TAO on coldkey**.

---

## Copy/Paste Blocks

**Install**

```bash
git clone https://github.com/fx-integral/metahash.git && cd metahash
python -m venv .venv && source .venv/bin/activate
pip install -U pip wheel uv && uv pip install -e .
cp .env.template .env
```

**Wallets**

```bash
btcli wallet new_coldkey --wallet.name team
btcli wallet new_hotkey --wallet.name team --wallet.hotkey miner1
```

**Register**

```bash
btcli subnets register --netuid 73 --wallet.name team --wallet.hotkey miner1
# or
btcli register --netuid 73 --wallet.name team --wallet.hotkey miner1
```

**Run miner**

```bash
python neurons/miner.py \
  --netuid 73 \
  --wallet.name team \
  --wallet.hotkey miner1 \
  --subtensor.network "ws://YOUR_NODE:9944" \
  --miner.bids.netuids 20 71 73 \
  --miner.bids.amounts 100.0 50.0 25.0 \
  --miner.bids.discounts 900 1200 2500 \
  --payment.validators HOTKEY_A HOTKEY_B \
  --logging.debug
```

**Unstake/Recycle**

```bash
btcli wallet overview --wallet.name team
btcli stake remove --wallet.name team --wallet.hotkey miner1 --amount 10
# or
btcli wallet recycle --wallet.name team --wallet.hotkey miner1 --amount 10
```

---

## Micro-Explanations (Short Words)

* **Coldkey**: your safe wallet. Holds TAO (liquid). Pays fees.
* **Hotkey**: your worker. Gets emissions (staked TAO).
* **α (alpha)**: the team token you sell (per subnet).
* **Discount bps**: big number form of percent. 100 bps = 1%.
* **Auction**: where you offer α at your max discount.
* **Win**: validator accepted your line(s). You must pay α in time.
* **[as,de]**: pay window in blocks. Pay only inside this.
* **Treasury**: safe address set by validator. Only pay there.
* **Burn**: underfill is destroyed (keeps math honest).
* **Weights → Emissions**: your reward → TAO on hotkey stake.
* **Unstake/Recycle**: move TAO from hotkey stake to coldkey.

---

## Troubleshooting

**“Insufficient balance … to register neuron”**

* Put a little **TAO on your coldkey** (fees), then retry register.

**“No Win”**

* Discount too tight? Try higher `discount_bps`.
* Subnet limits? Try smaller `α_amount`.

**“Paid but no weight”**

* Did you pay **within [as,de]**?
* Did you pay to the **right treasury** and **right subnet**?
* Any part you missed is **burned** → no weight for that part.

**“Only a few T per block land”**

* That’s okay. Use the whole window. Split into more txs.

**“Devnet vs Testnet vs Mainnet”**

* SN73 is a **mainnet subnet**. Connect to the right node URL.

**“Which wallet holds what?”**

* Fees & liquid TAO: **coldkey**.
* Emissions (staked): **hotkey**.

---

## Validator Notes (FYI)

* Validators publish **CID on-chain** and **full JSON to IPFS**. No catch-up.
* `STRICT_PER_SUBNET` can be on. Then each line must be paid per subnet.
* Reputation caps can limit per-coldkey fills.
* `AUCTION_BUDGET_ALPHA` drives how much α is accepted per epoch.

---

## Safety Checklist (Before You Bid)

* [ ] Coldkey funded for fees
* [ ] Hotkey registered on SN73
* [ ] α present on source hotkeys
* [ ] Treasury list confirmed (`metahash/treasuries.py`)
* [ ] Discount set to a max you accept
* [ ] You can send during the whole window

---

## Links

* **MetaHash (SN73) Repo**: `https://github.com/fx-integral/metahash`
* **Bittensor Docs**: `https://docs.bittensor.com/`
* **Coldkey/Hotkey Security**: `https://docs.learnbittensor.org/getting-started/coldkey-hotkey-security/`

---

> Want this README branded for your team (logo/colors) or split into separate guides (Quick Start vs Deep Dive)? Say the word and we’ll tailor it.
