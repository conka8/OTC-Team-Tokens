# MetaHash SN73 — Team OTC Sell Guide

> This page is for teams holding α (team tokens, e.g., SN20) who want to sell OTC-style through **Subnet 73**. Start with α in your wallet, end with TAO in your coldkey. Each step is literal, so your team can follow without guessing.

---

## Quick summary: what this guide does

* **Explains α → TAO via SN73 auction** in dumb-simple words.
* **Covers miner onboarding**: make wallet, register hotkey, run miner with bids, handle invoices.
* **Ends with TAO in your coldkey**: after emissions → unstake/recycle.

---

## Role snapshots

### Teams

* **Hold α in source hotkeys**.
* **Run SN73 miner** with bids `(subnet_id, α_amount, discount_bps)`.
* **Pay invoices** inside `[as,de]` window → treasury.
* **Unstake emissions** later → TAO on coldkey.

### Validators

* **Run auction → clearing → commitments → settlement**.
* **Whitelist treasuries** in `metahash/treasuries.py`.
* **Publish commitments** on-chain (CID) + payload to IPFS.

### Treasury

* **Receives α payments** during pay windows.
* **Only whitelisted addresses count**.

---

## Auction lifecycle (team perspective)

1. **Auction start (epoch e)** – Validator signals auction is open.
2. **Submit bids** – Your miner sends `(subnet_id, α_amount, discount_bps)`.
3. **Accept / Win** – Validator accepts if valid; sends `Win` with pay window `[as, de]`.
4. **Pay α (epoch e+1)** – Send α from your source hotkeys to treasury within window.
5. **Validator records** – Commitment on-chain (CID) + payload in IPFS.
6. **Settlement (epoch e+2)** – Validator checks α, burns underfill, sets weights, emissions → your hotkey.
7. **Unstake/Recycle** – Move TAO from hotkey stake → coldkey liquid.

---

## Step-by-step (literal)

### 1. Install

```bash
git clone https://github.com/fx-integral/metahash.git && cd metahash
python -m venv .venv && source .venv/bin/activate
pip install -U pip wheel uv && uv pip install -e .
cp .env.template .env
```

### 2. Wallets

```bash
btcli wallet new_coldkey --wallet.name team
btcli wallet new_hotkey --wallet.name team --wallet.hotkey miner1
```

Fund coldkey with a little TAO (for fees).

### 3. Register

```bash
btcli subnets register --netuid 73 --wallet.name team --wallet.hotkey miner1
```

### 4. Prepare α

* Put α (team tokens) on source hotkeys: `HOTKEY_A`, `HOTKEY_B`.
* These hotkeys will actually send α → treasury when you win.

### 5. Run Miner

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

### 6. Watch for `Win`

* Log shows **pay window** `[as,de]`.
* Treasury address appears (must match `treasuries.py`).

### 7. Pay α

* Pay within `[as,de]`.
* If you bid 100 α and only 3 α per block fit, chain fills slowly.
* Wrong treasury/subnet = ignored.

### 8. Settlement

* Validator checks payments.
* Burns underfill.
* Sets weights → emissions.

### 9. Get TAO

```bash
btcli wallet overview --wallet.name team
btcli stake remove --wallet.name team --wallet.hotkey miner1 --amount 10
# or
btcli wallet recycle --wallet.name team --wallet.hotkey miner1 --amount 10
```

Now TAO liquid in coldkey.

---

## How Auctions Work (detail for teams)

* **You choose maxDiscount (bps).** Example: 1000 bps = 10% max haircut.
* **Validator compares subnet baseline haircut + slippage** vs your maxDiscount.

  * If total ≤ maxDiscount → your bid can be accepted.
  * If total > maxDiscount → your bid is rejected.
* **Accepted lines → Win invoice.** You must pay α during pay window.
* **Payments come from your source hotkeys** (the ones listed under `--payment.validators`).
* **Partial fills possible**: if you bid 100T but chain only clears 3T per block, it takes many blocks to fully settle. You may fill slowly until window ends.
* **If all offers are at 20% discount but you only set 10% max** → you won’t sell. Your bid is too strict. You must either:

  * Wait until validators accept lower discounts.
  * Or raise your maxDiscount.
* **Burn safety**: if you underpay, the missing part is burned. You only get emissions for the part you really paid.

---

## Glossary (short words)

* **Coldkey** – safe wallet, holds TAO, pays fees.
* **Hotkey** – worker, gets emissions (staked TAO).
* **α** – team tokens you sell.
* **Discount bps** – 100 bps = 1% haircut max.
* **Win** – validator accepted, shows pay window.
* **[as,de]** – pay window in blocks.
* **Treasury** – whitelisted address.
* **Burn** – missing α destroyed.
* **Weights → Emissions** – your TAO reward.

---

## Checklist before bidding

* [ ] Coldkey funded for fees
* [ ] Hotkey registered on SN73
* [ ] α present on source hotkeys
* [ ] Treasury list confirmed (`metahash/treasuries.py`)
* [ ] Discount set (bps) at a level you accept
* [ ] Able to pay during window

---

## Links

* [MetaHash SN73 Repo](https://github.com/fx-integral/metahash)
* [Bittensor Docs](https://docs.bittensor.com/)
* [Miner Guide](../miner.md)
* [Validator Guide](../validator.md)
