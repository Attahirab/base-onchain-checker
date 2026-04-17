# base-onchain-checker
to check wallet reputation
pip install web3 requests
import requests
import time
from web3 import Web3
from web3.middleware import geth_poa_middleware
from datetime import datetime

# ================== CONFIG ==================
BASESCAN_API_KEY = "YOUR_BASESCAN_API_KEY_HERE"  # ← Replace with your free BaseScan API key from https://basescan.org/register

# Your details (pre-filled)
YOUR_ADDRESS = "0xf168202d5eab78a7da7b21f9cbf2a6f1c0bb263d"
YOUR_ENS = "Aablamido.eth"

BASE_RPC = "https://mainnet.base.org"
MAINNET_RPC = "https://ethereum.publicnode.com"

# ===========================================

def resolve_ens(ens_name: str) -> str:
    """Resolve ENS to address using public Mainnet RPC."""
    if not ens_name.endswith(".eth"):
        ens_name += ".eth"
    
    w3 = Web3(Web3.HTTPProvider(MAINNET_RPC))
    if not w3.is_connected():
        print("⚠️  Could not connect to ENS RPC, using address directly.")
        return None
    try:
        address = w3.ens.address(ens_name)
        if address:
            print(f"✅ Resolved {ens_name} → {address}")
            return address.lower()
        else:
            print(f"⚠️  ENS {ens_name} not found.")
            return None
    except Exception as e:
        print(f"⚠️  ENS resolution failed: {e}")
        return None

def get_base_activity(address: str):
    """Fetch transaction data from BaseScan API."""
    base_url = "https://api.basescan.org/api"
    params = {
        "apikey": BASESCAN_API_KEY,
        "module": "account",
        "action": "txlist",
        "address": address,
        "startblock": 0,
        "endblock": 99999999,
        "page": 1,
        "offset": 10000,
        "sort": "asc"
    }
    
    print("Fetching normal transactions...")
    response = requests.get(base_url, params=params, timeout=30)
    data = response.json()
    
    if data.get("status") != "1":
        raise Exception(f"BaseScan error: {data.get('message', 'Unknown error')}")
    
    txs = data.get("result", [])
    
    # Token transfers (ERC-20)
    print("Fetching token transfers...")
    token_params = params.copy()
    token_params["action"] = "tokentx"
    token_resp = requests.get(base_url, params=token_params, timeout=30)
    token_txs = token_resp.json().get("result", []) if token_resp.json().get("status") == "1" else []
    
    # NFT transfers
    print("Fetching NFT transfers...")
    nft_params = params.copy()
    nft_params["action"] = "tokennfttx"
    nft_resp = requests.get(base_url, params=nft_params, timeout=30)
    nft_txs = nft_resp.json().get("result", []) if nft_resp.json().get("status") == "1" else []
    
    return txs, token_txs, nft_txs

def calculate_onchain_score(txs, token_txs, nft_txs):
    """Transparent onchain activity score (0-100)"""
    if not txs:
        return 0, {"Note": "No transactions found on Base"}
    
    total_txs = len(txs)
    unique_contracts = len({tx["to"] for tx in txs if tx.get("to")}) + len({tx["from"] for tx in txs if tx.get("from")})
    unique_days = len({datetime.fromtimestamp(int(tx["timeStamp"])).date() for tx in txs})
    span_days = (int(txs[-1]["timeStamp"]) - int(txs[0]["timeStamp"])) / 86400 if len(txs) > 1 else 0
    token_transfers = len(token_txs)
    nft_transfers = len(nft_txs)
    
    # Scoring components
    tx_score = min(40, total_txs * 0.4)
    diversity_score = min(25, unique_contracts * 1.2)
    consistency_score = min(20, unique_days * 1.8)
    longevity_score = min(10, min(span_days / 30, 10))
    defi_nft_score = min(5, token_transfers * 0.08) + min(5, nft_transfers * 0.15)
    
    total_score = round(tx_score + diversity_score + consistency_score + longevity_score + defi_nft_score)
    
    breakdown = {
        "Total Transactions on Base": total_txs,
        "Unique Contracts Interacted": unique_contracts,
        "Unique Active Days": unique_days,
        "Activity Span (days)": round(span_days, 1),
        "Token Transfers": token_transfers,
        "NFT Transfers": nft_transfers,
        "Score Breakdown": {
            "Transactions": round(tx_score, 1),
            "Diversity": round(diversity_score, 1),
            "Consistency": round(consistency_score, 1),
            "Longevity": round(longevity_score, 1),
            "DeFi & NFT Activity": round(defi_nft_score, 1)
        }
    }
    return total_score, breakdown

# ================== MAIN ==================
if __name__ == "__main__":
    print("🔹 BASE Onchain Activity Score Checker 🔹\n")
    print(f"Your default wallet: {YOUR_ADDRESS}")
    print(f"Your ENS: {YOUR_ENS}\n")
    
    user_input = input("Enter ENS or address (or press Enter to use your wallet): ").strip()
    
    if not user_input:
        # Use your details
        address = YOUR_ADDRESS
        ens = YOUR_ENS
        print(f"Using your wallet → {address} ({ens})")
    elif user_input.endswith(".eth") or "." in user_input:
        ens = user_input
        address = resolve_ens(ens)
        if not address:
            address = YOUR_ADDRESS  # fallback
    else:
        address = user_input.lower()
        if not address.startswith("0x"):
            address = "0x" + address
        ens = "Custom Address"
    
    if len(address) != 42 or not address.startswith("0x"):
        print("❌ Invalid address format.")
        exit(1)
    
    print(f"\n🔍 Fetching onchain activity for {address} on Base...\n")
    
    try:
        txs, token_txs, nft_txs = get_base_activity(address)
        score, breakdown = calculate_onchain_score(txs, token_txs, nft_txs)
        
        print("\n" + "="*60)
        print(f"🎯 YOUR BASE ONCHAIN ACTIVITY SCORE: {score}/100")
        print("="*60)
        
        for key, value in breakdown.items():
            if isinstance(value, dict):
                print(f"\n{key}:")
                for sub_key, sub_value in value.items():
                    print(f"   • {sub_key}: {sub_value}")
            else:
                print(f"{key}: {value}")
        
        print("\n💡 This is a transparent open-source approximation based on Base activity.")
        print("   The official Coinbase Onchain Reputation score (onchainscore.xyz) may vary slightly.")
        
    except Exception as e:
        print(f"❌ Error: {e}")
        print("\nTips:")
        print("• Make sure your BASESCAN_API_KEY is correct")
        print("• BaseScan free tier has rate limits – wait a bit and try again")
        print("• You can increase 'offset' if you have >10k transactions")
