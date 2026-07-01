# base432vii
0xC1fC064c2EF5F8186c8467296764008C2E84F12B
import time
from collections import defaultdict
from statistics import mean
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

WINDOW_BLOCKS = 20
MIN_PROJECTS = 5

ZERO = "0x0000000000000000000000000000000000000000"


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


def main():

    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise RuntimeError("Cannot connect")

    print("Connected to Base")
    print("Profiling wallet behavior...\n")

    last_block = w3.eth.block_number

    wallet_projects = defaultdict(set)
    wallet_first_seen = defaultdict(list)
    wallet_sent = defaultdict(int)
    wallet_received = defaultdict(int)

    while True:

        try:

            current_block = w3.eth.block_number

            if current_block >= last_block + WINDOW_BLOCKS:

                logs = w3.eth.get_logs({
                    "fromBlock": current_block - WINDOW_BLOCKS,
                    "toBlock": current_block,
                    "topics": [TRANSFER_TOPIC]
                })

                for log in logs:

                    token = log["address"]

                    sender = decode_address(
                        log["topics"][1]
                    )

                    receiver = decode_address(
                        log["topics"][2]
                    )

                    block = log["blockNumber"]

                    if sender != ZERO:

                        wallet_projects[sender].add(token)
                        wallet_sent[sender] += 1

                    if receiver != ZERO:

                        wallet_projects[receiver].add(token)
                        wallet_received[receiver] += 1

                        wallet_first_seen[
                            receiver
                        ].append(block)

                print(
                    f"\nBlocks "
                    f"{current_block-WINDOW_BLOCKS}"
                    f" -> {current_block}"
                )

                for wallet in wallet_projects:

                    projects = len(
                        wallet_projects[wallet]
                    )

                    if projects < MIN_PROJECTS:
                        continue

                    avg_block = mean(
                        wallet_first_seen[wallet]
                    )

                    print("📊 Wallet Profile")
                    print("Wallet:", wallet)
                    print(
                        "Projects:",
                        projects
                    )
                    print(
                        "Transfers In:",
                        wallet_received[wallet]
                    )
                    print(
                        "Transfers Out:",
                        wallet_sent[wallet]
                    )
                    print(
                        "Average First Seen Block:",
                        round(avg_block, 1)
                    )
                    print()

                last_block = current_block

            time.sleep(3)

        except Exception as e:

            print("Error:", e)
            time.sleep(5)


if __name__ == "__main__":
    main()
