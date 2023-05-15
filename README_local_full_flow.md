

## Instalation

WARNING:  Highly WIP, most likely you will get errors or branches are out of date, etc.. 

This tutorial uses the private keys from barge:
 - OPF_DEPLOYER_PRIVATE_KEY:  `0xc594c6e5def4bab63ac29eed19a134c130388f74f019bc74b8f4389df2837a58`  - contracts owner, ocean token owner
 - PREDICTOOR_PRIVATE_KEY: `0xef4b441145c1d0f3b4bc6d61d29f5c6e502359481152f869247c7a4244d45209`  - predictoor

Create four terminals, call them `barge`, `ocean.py`, `pd-predictoor`, `pd-trueval`.


### 1. Barge terminal

```bash
export ADDRESS_FILE="${HOME}/.ocean/ocean-contracts/artifacts/address.json"
git clone https://github.com/oceanprotocol/barge.git
cd barge
git checkout predictoor

./start_ocean.sh --predictoor
```

This will start barge with a custom version of ganache (auto-mine a block every 12 sec), contracts (predictoor), subgraph (predictoor)
WARNING:   Barge will start more slowly, deploying contracts takes a couple of minutes.  Watch the output to know when to proceed further or check if file "/ocean-contracts/artifacts/ready" exists

Go to next step when barge is ready and contracts are deployed

## 2. Ocean.py  - create a template3 token

Since there is no easy way now to create a template3 datatoken, we will use ocean.py.  This flow will be replaced

```bash
export ADDRESS_FILE="${HOME}/.ocean/ocean-contracts/artifacts/address.json"
export OPF_DEPLOYER_PRIVATE_KEY="0xc594c6e5def4bab63ac29eed19a134c130388f74f019bc74b8f4389df2837a58"
export PREDICTOOR_PRIVATE_KEY="0xef4b441145c1d0f3b4bc6d61d29f5c6e502359481152f869247c7a4244d45209"
git clone https://github.com/oceanprotocol/ocean.py
cd ocean.py
git checkout predictoor-with-barge
```


Install requirments, activate venv, etc and open a pyton console in which paste the following:

```python
import brownie
from brownie.network import accounts as br_accounts
import os
from ocean_lib.ocean.ocean import Ocean
from ocean_lib.models.datatoken_base import DatatokenBase
from ocean_lib.web3_internal.utils import connect_to_network
from ocean_lib.ocean.util import from_wei, to_wei
from ocean_lib.example_config import get_config_dict
from ocean_lib.web3_internal.constants import ZERO_ADDRESS, MAX_UINT256
# add accounts
deployer = br_accounts.add(os.getenv("OPF_DEPLOYER_PRIVATE_KEY"))
predictoor = br_accounts.add(os.getenv("PREDICTOOR_PRIVATE_KEY"))
connect_to_network("development")
ADDRESS_FILE = "~/.ocean/ocean-contracts/artifacts/address.json"
address_file = os.path.expanduser(ADDRESS_FILE)
print(f"Load contracts from address_file: {address_file}")
config = get_config_dict("development")
config["ADDRESS_FILE"] = address_file
ocean = Ocean(config)
OCEAN = ocean.OCEAN_token

# transfer ocean tokens to staker
OCEAN.transfer(predictoor_address, to_wei(2000.0), {"from": deployer})

#create NFT
data_nft = ocean.data_nft_factory.create({"from": deployer}, "DN", "DN")
S_PER_MIN = 60
S_PER_HOUR = 60 * 60
s_per_block = 2  # depends on the chain
s_per_epoch = 5 * S_PER_MIN
s_per_subscription = 24 * S_PER_HOUR
min_predns_for_payout = 3  # ideally, 100+
stake_token = OCEAN
DT_price = 2

#create template3
initial_list = data_nft.getTokensList()
data_nft.createERC20(
        3,
        ["ETH-USDT", "ETH-USDT"],
        [deployer.address, deployer.address, deployer.address, OCEAN.address, OCEAN.address],
        [MAX_UINT256, 0, s_per_block, s_per_epoch, s_per_subscription, 30],
        [],
        {"from": deployer},
    )
new_elements = [
        item for item in data_nft.getTokensList() if item not in initial_list
    ]
assert len(new_elements) == 1, "new datatoken has no address"
DT = DatatokenBase.get_typed(config, new_elements[0])
DT.setup_exchange({"from": deployer}, to_wei(DT_price))
```


### 3. pdr-trueval

```bash
git clone https://github.com/oceanprotocol/pdr-trueval.git
cd pdr-trueval
pip install -r requirements.txt
export ADDRESS_FILE="${HOME}/.ocean/ocean-contracts/artifacts/address.json"
export RPC_URL=http://127.0.0.1:8545
export SUBGRAPH_URL="http://172.15.0.15:8000/subgraphs/name/oceanprotocol/ocean-subgraph"
export PRIVATE_KEY="0xc594c6e5def4bab63ac29eed19a134c130388f74f019bc74b8f4389df2837a58"
python3 main.py
```


### 4. pdr-predictoor

```bash
git clone https://github.com/oceanprotocol/pdr-predictoor.git
cd pdr-predictoor
pip install -r requirements.txt
export ADDRESS_FILE="${HOME}/.ocean/ocean-contracts/artifacts/address.json"
export RPC_URL=http://127.0.0.1:8545
export SUBGRAPH_URL="http://172.15.0.15:8000/subgraphs/name/oceanprotocol/ocean-subgraph"
export PRIVATE_KEY="0xef4b441145c1d0f3b4bc6d61d29f5c6e502359481152f869247c7a4244d45209"
python3 main.py
```

## Relax & watch

Now, watch as pdr-predictoor is submiting random predictions , and pdr-trueval submits random true_vals for each epoch

You can query (subgraph)[http://172.15.0.15:8000/subgraphs/name/oceanprotocol/ocean-subgraph/graphql] and see populated data  (PR)[https://github.com/oceanprotocol/ocean-subgraph/pull/678] here for entities to be queried


## Next Steps
    Customize (pdr-predictoor)[https://github.com/oceanprotocol/pdr-predictoor] and (pdr-trueval)[https://github.com/oceanprotocol/pdr-trueval] not to submit random values, but actually use real data