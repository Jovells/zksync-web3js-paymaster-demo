# Building a zkSync dApp with Stablecoin Gas Payments using Web3.js

This tutorial will guide you through building a decentralized application that allows users to pay transaction gas fees using stablecoins instead of ETH on zkSync. We'll use Web3.js and zkSync's Paymaster feature to achieve this.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Setup](#project-setup)
3. [Smart Contract Setup](#smart-contract-setup)
4. [Paymaster Integration](#paymaster-integration)
5. [User Interface](#user-interface)
6. [Testing](#testing)

## Prerequisites

- Node.js v14 or later
- Basic knowledge of React and TypeScript
- Understanding of Web3 concepts
- MetaMask wallet
- Basic understanding of smart contracts

## Project Setup

1. Create a new React TypeScript project:

```bash
npx create-react-app zksync-paymaster-demo --template typescript
cd zksync-paymaster-demo
```

2. Install required dependencies:

```bash
npm install web3 web3-plugin-zksync zksync-web3-contract-paymaster-plugin react-router-dom react-hot-toast
```

3. Configure your `package.json`:

```json
{
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "web3": "^4.12.1",
    "web3-plugin-zksync": "^1.0.6",
    "zksync-web3-contract-paymaster-plugin": "^1.0.6"
  }
}
```

## Smart Contract Setup

Store your contract addresses in a constants file:

```typescript
export const MUSDT_ADDRESS = "0xff68f7561562C1F24A317d939B46741F76c4Ef55";
export const MUSDT_PAYMASTER_ADDRESS = "0xfA7Adc05E56893df8ecE1F63E4dB1db7767146f4";
export const STORE_CONTRACT_ADDRESS = "0xEc969112DB5440c954CB60B4Bbd1159673eeE4C3";
```

## Paymaster Integration

### Understanding Paymasters in zkSync

Before diving into the implementation, it's important to understand what a Paymaster is and how it works in zkSync.

A Paymaster is a smart contract in zkSync that enables gas abstraction - allowing users to pay transaction fees with tokens other than ETH. This is a powerful feature that improves user experience by:
- Letting users pay gas fees in ERC20 tokens
- Enabling sponsored transactions where a third party pays the gas
- Supporting more complex gas payment scenarios

The `zksync-web3-contract-paymaster-plugin` uses a default testnet paymaster if no paymaster address is provided. This paymaster is used for testing purposes only and may not work sometimes (if it runs out of funds). In this tutorial we are using a custom made paymaster which accepts a custom token as referenced above: `MUSDT_PAYMASTER_ADDRESS` and `MUSDT_ADDRESS`. It is recommended to deploy your own paymaster and fund it.

For mainnet deployment, you'll need to use a paymaster on mainnet which accepts the token you want your users to pay in or create and deploy your own paymaster contract. A comprehensive guide on creating a custom ERC20 paymaster can be found in the [zkSync ERC20 Paymaster Tutorial](https://code.zksync.io/tutorials/erc20-paymaster).

### Implementation

1. Create a custom hook for Paymaster interactions:

```typescript
import { useState, useEffect, useCallback } from 'react';
import Web3 from 'web3';
import { getPaymasterParams, types, Web3ZKsyncL2, ZKsyncPlugin } from 'web3-plugin-zksync';
import ZkSyncContractPaymasterPlugin from "zksync-web3-contract-paymaster-plugin";
import { MUSDT_ADDRESS } from './constants';

const usePaymasterAsync = (contractAddress: string, contractAbi: any, _paymasterAddress?: string) => {
    // Note: _paymasterAddress is optional - if not provided, the plugin will use testnet paymaster
    const [web3, setWeb3] = useState<Web3>();
    const [plugin, setPlugin] = useState<ZkSyncContractPaymasterPlugin>();

    // Initialize Web3 and register the required plugins
    useEffect(() => {
        const initializeWeb3 = async () => {
            if (typeof window.ethereum !== 'undefined') {
                // Create Web3 instance
                const web3 = new Web3(window.ethereum);
                
                // Initialize zkSync L2 provider for Sepolia testnet
                const l2 = Web3ZKsyncL2.initWithDefaultProvider(types.Network.Sepolia);
                
                // Create instance of the Paymaster plugin
                const plugin = new ZkSyncContractPaymasterPlugin(window.ethereum);
                
                // Register both plugins with Web3
                web3.registerPlugin(plugin);
                web3.registerPlugin(new ZKsyncPlugin(l2));
                
                setPlugin(plugin);
                setWeb3(web3);
            }
        };

        initializeWeb3();
    }, []);

    // This function handles contract interactions with Paymaster integration
    const writeContractWithPaymaster = useCallback(async (
        { functionName, args }: { functionName: string; args: any[] }
    ) => {
        if (!web3 || !plugin) return;
        
        return await plugin.write(contractAddress, contractAbi, {
            methodName: functionName,
            args: args,
            from: await web3.eth.getAccounts().then(a => a[0]),
            customData: {
                // Gas per pubdata is a zkSync-specific parameter
                gasPerPubdata: 50000,
                // Configure Paymaster parameters for gas abstraction
                paymasterParams: getPaymasterParams(_paymasterAddress, {
                    type: "ApprovalBased",
                    minimalAllowance: 10,
                    token: MUSDT_ADDRESS,
                    innerInput: new Uint8Array(),
                })
            }
        });
    }, [web3, plugin, contractAddress]);

    return { writeContractWithPaymaster };
};
```

The `usePaymasterAsync` hook does several important things:
1. Initializes Web3 with zkSync plugins
2. Provides a function to interact with any smart contract while using the Paymaster
3. Handles the configuration of Paymaster parameters for gas fee abstraction

2. Implement the purchase function with Paymaster support:

This example shows how to use the Paymaster hook with any smart contract. In this case, we're using a marketplace contract that takes a product ID and quantity, but you can adapt this pattern for any contract function:

```typescript
const buyProductWithPayMaster = async (product: Product, itemQty: number) => {
    try {
        // IMPORTANT: First approve the spending of tokens
        // This is required for any ERC20 token interaction
        const approvalTx = await approveWithPaymasterAsync(
            { 
                functionName: 'approve',
                args: [STORE_CONTRACT_ADDRESS, product.price]
            }
        );
        
        // Wait for approval transaction to be confirmed
        await approvalTx.wait();

        // Then make the actual contract call
        // This could be any function on any contract
        const tx = await buyWithPaymasterAsync(
            {
                functionName: 'purchaseProduct', // Your contract function name
                args: [product.id, itemQty]     // Your function arguments
            }
        );
        
        return { tx, product };
    } catch (error) {
        console.error('Error buying product with Paymaster', error);
        throw error;
    }
};

// Example of how this is used in a component
const Products = ({ account, web3 }) => {
    // Initialize the Paymaster hooks for both token and marketplace contracts
    const { writeContractWithPaymaster: approveWithPaymasterAsync } = usePaymasterAsync(
        MUSDT_ADDRESS,    // Token contract
        stablecoinAbi, 
        MUSDT_PAYMASTER_ADDRESS
    );
    
    const { writeContractWithPaymaster: buyWithPaymasterAsync } = usePaymasterAsync(
        STORE_CONTRACT_ADDRESS,  // Your contract
        deworldAbi, 
        MUSDT_PAYMASTER_ADDRESS
    );

    // Function to handle the purchase
    async function buy(product: Product, quantity: number) {
        if (!web3 || !account) {
            toast.error('Wallet not connected');
            return;
        }

        const id = toast.loading(`Purchasing ${product.name}... and paying gas fees with mUSDT`);
        try {
            const val = await buyProductWithPayMaster(product, quantity);
            if (!val) return;
            
            const { tx } = val;
            toast.success(`Purchase successful. Gas paid with mUSDT`, { id });
        } catch (error) {
            toast.error('Error purchasing product: ' + error, { id });
        }
    }

    return (
        // Your UI components
    );
};
```

The key points to understand are:

1. **Token Approval**: Before any ERC20 token transaction, you must approve the spending amount. This is standard for all ERC20 tokens.

2. **Contract Interaction**: The example shows a purchase function, but you can use this pattern with any smart contract function. Just modify:
   - The contract address
   - The contract ABI
   - The function name
   - The function arguments

3. **Gas Payment**: The Paymaster handles converting your token payment into gas fees automatically.

## User Interface

Create a simple interface for users to interact with:

```typescript
const Landing = ({ account, balance, web3 }) => {
    const mintStableCoin = async () => {
        if (!web3) {
            toast.error('Please connect your wallet');
            return initializeWeb3();
        }
        try {
            const contract = new web3.eth.Contract(stablecoinAbi, MUSDT_ADDRESS);
            await contract.methods.mint().send({
                from: account as string,
            });
            toast.success('Stablecoin minted successfully!');
            fetchBalance(web3);
        } catch (error) {
            console.log(error);
            toast.error('Failed to mint stablecoin');
        }
    };

    return (
        <div>
            <h1>zkSync Paymaster Demo</h1>
            <button onClick={mintStableCoin}>Mint mUSDT</button>
            <div>Balance: {balance} mUSDT</div>
            {/* Add more UI elements */}
        </div>
    );
};
```

## Testing

1. Configure MetaMask for zkSync Sepolia testnet:
   - Network Name: zkSync Sepolia
   - RPC URL: https://sepolia.era.zksync.dev
   - Chain ID: 300
   - Currency Symbol: ETH

2. Get test ETH from the zkSync Sepolia faucet

3. Test the following flows:
   - Minting mUSDT
   - Approving mUSDT spending
   - Making purchases with Paymaster
   - Viewing transaction history

## Key Considerations

1. **Gas Estimation**: The Paymaster needs to estimate gas costs accurately to ensure sufficient stablecoin balance.

2. **Error Handling**: Implement robust error handling for failed transactions.

3. **User Experience**: Provide clear feedback during transaction processing.

4. **Security**: Ensure proper approval mechanisms are in place for token spending.

## Conclusion

This tutorial demonstrated how to build a dApp that leverages zkSync's Paymaster feature to enable stablecoin gas payments. The key components are:

- Integration with zkSync's L2 network using the Paymaster plugin
- Smart contract interactions for token approvals and purchases
- User interface for seamless interaction

You can view a live which demo at [Web3js ZkSync Plugin Demo](https://web3js-zksync-plugin.vercel.app/). 