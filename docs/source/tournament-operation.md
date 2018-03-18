# Tournament Operation

Towards the end of 2017, Gnosis hosted a prediction market tournament called [Olympia](https://blog.gnosis.pm/announcing-gnosis-olympia-5fb7e16dd259). It combined [GnosisDB](https://github.com/gnosis/gnosisdb), the [core smart contracts](https://github.com/gnosis/gnosis-contracts), and this library in the context of a [user interface](https://github.com/gnosis/gnosis-management) we've been developing (we've also created a [separate repository](https://github.com/gnosis/olympia-interface) for the interface used in the original tournament run, but the work done there is slated to be refactored into a feature flag for the generic interface).

This tutorial will detail the configuration and operation of such a tournament.

## Smart Contracts

First, you will want to deploy contracts necessary for the tournament. These contracts can be found in the [`olympia-token`](https://github.com/gnosis/olympia-token) repository. It will likely be the case that you will want to fork this repository in order to tweak it for your own purposes (you will know if that is not the case), so hit that fork button. After forking it, clone your fork and open the folder in a terminal:

```sh
# this is the SSH version; feel free to use HTTPS if you prefer
git clone git@github.com:your-github-handle/olympia-token.git
cd olympia-token
```

Have NPM install the dependencies for the repository:

```sh
npm i
```

### Token Contracts

After the dependencies have been installed, take a look at the [OlympiaToken contract](https://github.com/gnosis/olympia-token/blob/master/contracts/OlympiaToken.sol). You will see that this contract is more or less functionally equivalent to the [PlayToken contract](https://github.com/gnosis/olympia-token/blob/master/contracts/PlayToken.sol), except that the optional [ERC20](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md) fields `name`, `symbol`, and `decimals` have also been specified.

Let's turn our attention now to the [PlayToken contract](https://github.com/gnosis/olympia-token/blob/master/contracts/PlayToken.sol). Beyond the bare minimum necessary to implement the ERC20 interface sanely, this contract also implements two other mechanics: issuing new tokens to addresses, and determining a whitelist for token transfers.

The ability to issue new tokens to addresses, or minting tokens, is restricted to the creator of the contract instance. This action can be performed with a call to the `issue` function, which takes an array of recipient addresses and credits them a newly minted amount of token. The token issuance may, for example, be tied into a registration process for the tournament.

The creator of the contract instance is also required to set up a whitelist for token transfers. This prevents arbitrary transfers of token value from one account to another. PlayTokens can be transferred only to or from the whitelisted addresses. You may use this mechanic to ensure that users can only trade on authorized markets in the tournament.

Due to the mechanics above, it is crucial for the integrity of the tournament that the private key used to deploy the token contract is guarded carefully, and that actions taken by the contract creator do not undermine the faith of the tournament players.

You may implement other token mechanics as you see fit, but that is beyond the scope of this document. 

If you do not wish to change the token mechanics, but you do wish to change the name and symbol of the token used by your tournament, then you may do so by renaming and modifying the [OlympiaToken contract](https://github.com/gnosis/olympia-token/blob/master/contracts/OlympiaToken.sol) accordingly. For example:

```sol
pragma solidity 0.4.15;

import "./PlayToken.sol";

contract BigToken is PlayToken {
    /*
     *  Constants
     */
    string public constant name = "Big Token";
    string public constant symbol = "BIG";
    uint8 public constant decimals = 18;
}
```

If you do so, you will also want to change the deployment script `migrations/2_deploy_contracts.js` to reflect the new token which is being deployed:

```js
const Math = artifacts.require('Math')
const BigToken = artifacts.require('BigToken')

module.exports = function(deployer) {
    deployer.deploy(Math)
    deployer.link(Math, BigToken)
    deployer.deploy(BigToken)
}
```

You may also consider changing the test suite in `test/olympia.js` so the test suite continues to pass:

```diff
diff --git a/test/olympia.js b/test/olympia.js
index 7ae4d15..2c33ff4 100644
--- a/test/olympia.js
+++ b/test/olympia.js
@@ -6,7 +6,7 @@ const _ = require('lodash')
 const { wait } = require('@digix/tempo')(web3);
 
 const MathLib = artifacts.require('Math')
-const OlympiaToken = artifacts.require('OlympiaToken')
+const BigToken = artifacts.require('BigToken')
 const PlayToken = artifacts.require('PlayToken')
 PlayToken.link(MathLib)
 const AddressRegistry = artifacts.require('AddressRegistry')
@@ -26,17 +26,17 @@ async function throwUnlessRejects(q) {
 
 const getBlock = util.promisify(web3.eth.getBlock.bind(web3.eth))
 
-contract('OlympiaToken', function(accounts) {
-    let olympiaToken
+contract('BigToken', function(accounts) {
+    let bigToken
 
     before(async () => {
-        olympiaToken = await OlympiaToken.deployed()
+        bigToken = await BigToken.deployed()
     })
 
     it('should have the right name, symbol, and decimals', async () => {
-        assert.equal(await olympiaToken.name(), 'Olympia Token')
-        assert.equal(await olympiaToken.symbol(), 'OLY')
-        assert.equal(await olympiaToken.decimals(), 18)
+        assert.equal(await bigToken.name(), 'Big Token')
+        assert.equal(await bigToken.symbol(), 'BIG')
+        assert.equal(await bigToken.decimals(), 18)
     })
 })
 
```

### Deployment to a Public Network

The original Olympia was conducted primarily on the [Rinkeby test network](https://www.rinkeby.io/).
