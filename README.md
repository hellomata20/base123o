# base123oimport time
from collections import defaultdict
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

WINDOW_BLOCKS = 20
HOLDER_THRESHOLD = 25


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


def main():
    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise RuntimeError("Cannot connect to Base RPC")

    print("Connected to Base")
    print("Detecting rapid token holder growth...\n")

    last_block = w3.eth.block_number

    while True:
        try:
            current_block = w3.eth.block_number

            if current_block >= last_block + WINDOW_BLOCKS:

                from_block = current_block - WINDOW_BLOCKS
                to_block = current_block

                logs = w3.eth.get_logs({
                    "fromBlock": from_block,
                    "toBlock": to_block,
                    "topics": [TRANSFER_TOPIC]
                })

                token_holders = defaultdict(set)

                for log in logs:

                    token = log["address"]
                    to_addr = decode_address(log["topics"][2])

                    # ignore burn/mint zero address
                    if to_addr != "0x0000000000000000000000000000000000000000":
                        token_holders[token].add(to_addr)

                print(f"\nBlocks {from_block} → {to_block}")

                for token, holders in token_holders.items():

                    if len(holders) >= HOLDER_THRESHOLD:
                        print("🚀 Fast Holder Growth")
                        print("Token:", token)
                        print("New holders:", len(holders))
                        print()

                last_block = current_block

            time.sleep(3)

        except Exception as e:
            print("Error:", e)
            time.sleep(5)


if __name__ == "__main__":
    main()
