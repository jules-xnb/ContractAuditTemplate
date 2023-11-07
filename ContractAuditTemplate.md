# Review XXX-contracts du 13/10/2023

### Résumé

La review du code du repo [XXX-contracts] a été faite depuis la branche [`contract/vesting`], au commit `XXX`.

La quantité de vulnérabilités détectées est la suivante :

| Sévérité | Détécté |
|------------------|----|
| 🚨 Critical      |  1 |
| 🔴 High          |  0 |
| 🟡 Medium        |  4 |
| 🟢 Low           |  7 |
| 💬 Informational |  3 |
| 🛠️ Optimization  |  4 |
| ❓ Question      |  2 |

-------

## XXX-OPT01: Utilisation d'un `require` redondant
**Severité:** `Optimization`
**Target:** `XXX.sol`

##### Description

La vérification du rôle `BURNER_ROLE` dans la fonction `burn()` se fait à travers un `require()` qui confirme si l'appelant dans le contexte possède le bon rôle.

```solidity
function burn(uint256 amount) public {
        // Vérification que l'appelant a le rôle de brûleur
        require(hasRole(BURNER_ROLE, msg.sender), "Caller is not a burner");

        // Burn des tokens
        _burn(msg.sender, amount);
    }
```

Ce code est redondant et peut être simplifié avec le modifier  `onlyRole()` de la librairie `AccessControl`.

##### Recommandation

Supprimer le [`require` de la ligne 41] et ajouter le modifier `onlyRole(BURNER_ROLE)` dans la signature de la fonction.

--------

## XXX-OPT02: Variable `immutable` non initialisée dans le `constructor`
**Severité:** `Optimization`
**Target:** `Vesting.sol`

##### Description

La variable `cliffPeriod` définie comme `immutable` n'est pas initialisée dans le `constructor` du contrat `Vesting.sol`.

```solidity
uint256 public immutable cliffPeriod = 90 days;
```

##### Recommandation

Remplacer `immutable` par `constant` [à la ligne 37].

--------

## XXX-OPT03: Variable `immutable`
**Severité:** `Optimization`
**Target:** `Vesting.sol`

##### Description

La variable `vestedToken` pourrait être définie comme `immutable` puisqu'elle est uniquement initialisée dans le `constructor` du contrat `Vesting.sol`.
```solidity
IERC20 public vestedToken;
```

##### Recommandation

Ajouter `immutable` [à la ligne 47].

--------

## XXX-OPT04: Appel interne inutile
**Severité:** `Optimization`
**Target:** `VestingAggregator.sol`

##### Description

La fonction `removeVesting()` fait appel à la fonction interne `vestingAddresses.contains()` afin de vérifier si le contrat de vesting à supprimer du registre des vestings est bien présent ou non avant de le supprimer.
Cependant, cette vérification est rendonante puisqu'elle est déjà effectué dans la fonction de suppression `vestingAddresses.remove()`.

```solidity
require(
    vestingAddresses.contains(vestingContract),
    "Vesting contract not found"
);
vestingAddresses.remove(vestingContract);
```

##### Recommandation

Remplacer [le contenu de la fonction `removeVesting()`] par:

```solidity
require(
    vestingAddresses.remove(vestingContract),
    "Vesting contract not found"
);
```

Dans le cas où le `vestingContract` à supprimer n'existe pas dans le registre, `vestingAddresses.remove()` retournera une valeur booléene `false` et le `require()` effectuera un `revert`.

--------

## XXX-INF01: Utilisation du `_setupRole` déprécié
**Severité:** `Informational`
**Target:** `XXX.sol`

##### Description

La fonction `_setupRole()` du contrat OpenZeppelin `AccessControl` est maintenant dépréciée.

```solidity
_setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
```

##### Recommandation

Remplacer la fonction `_setupRole(DEFAULT_ADMIN_ROLE, msg.sender);` à [la ligne 35] par `_grantRole(DEFAULT_ADMIN_ROLE, msg.sender);`.

--------

## XXX-INF02: Variable initialisée deux fois
**Severité:** `Informational`
**Target:** `Vesting.sol`

##### Description

La variable `totalVestingDuration` est configurée deux fois: 
La première fois dans le bytecode du contrat au moment du déploiement avec une valeure par défaut à `365 days`, et une deuxième fois dans le `constructor` du contrat.

```solidity
uint256 public totalVestingDuration = 365 days;
...
constructor(address vestedTokenAddress, uint256 _totalVestingDuration) {
        vestedToken = IERC20(vestedTokenAddress);
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        
        if (_totalVestingDuration > 0) {
            totalVestingDuration = _totalVestingDuration;
        }
    }
```

##### Recommandation

Il est recommandé de choisir une manière claire de définir la variable `totalVestingDuration` à [la ligne 41](totalVestingDuration).

--------

## XXX-INF03: Entrypoint `transferOwnership()` non nécessaire
**Severité:** `Informational`
**Target:** `Vesting.sol`

##### Description

La fonction `transferOwnership(address newOwner)` permet de changer le propriétaire du contrat. Pour ce faire, elle ajoute un nouveau administrateur et supprime l'administrateur actuel (l'adresse en contexte de l'appel). Cependant, bien que la fonction facilite l'appel de ces deux méthodes en une seule exécution, il est aussi possible d'obtenir le même résultat en appelant les fonctions publiques héritées par `AccessControl`.

```solidity
function transferOwnership(address newOwner)
        external
        onlyRole(DEFAULT_ADMIN_ROLE)
    {
        _grantRole(DEFAULT_ADMIN_ROLE, newOwner);
        _revokeRole(DEFAULT_ADMIN_ROLE, msg.sender);
    }
```

##### Recommandation

[L'entrypoint `transferOwnership(address newOwner)` à la ligne 105] peut être supprimé du code.
Lors de la passation d'administration du contrat, il faudra alors effectuer deux appels distincts: 
d'abord un appel à la fonction `grantRole(DEFAULT_ADMIN_ROLE, newOwner)` pour ajouter le nouvel administrateur, puis un second appel pour abandonner le rôle d'administrateur avec `renounceRole(DEFAULT_ADMIN_ROLE, msg.sender)`.

--------

## XXX-L01: Variables non utilisés
**Severité:** `Low`
**Target:** `XXX.sol`

##### Description

Des variables non utilisées sont présentes dans le code et peuvent prêter à confusion.
```solidity
uint8 public immutable burnPercentage = 2;
uint8 public immutable operationalFundPercentage = 2;
uint8 public immutable communityRewardTreasuryPercentage = 96;
```

##### Recommandation

Supprimer l'ensemble des variables inutilisés de [la ligne 14 à la ligne 16].

--------

## XXX-L02: Events non utilisés
**Severité:** `Low`
**Target:** `XXX.sol`

##### Description

Des `events` non utilisés sont présents dans le code et peuvent prêter à confusion.

```solidity
event BurnerRoleGranted(address indexed burnerAddress);
event BurnerRoleRevoked(address indexed burnerAddress);
event OwnerRoleGranted(address indexed ownerAddress);
event OwnerRoleRevoked(address indexed ownerAddress);
event TokensDispersed(
    address indexed burnAddress,
    address indexed operationalFundAddress,
    address indexed communityRewardTreasuryAddress,
    uint256 totalBalance
);
```

##### Recommandation

Supprimer l'ensemble des `events` inutilisés de [la ligne 19 à la ligne 28].

--------

## XXX-L03: Fonction inutile
**Severité:** `Low`
**Target:** `XXX.sol`

##### Description

La fonction `getContractBalance()` permet de récupérer la balance de tokens ERC-20 détenus dans le contrat. Cependant, il est déjà possible d'effectuer cette action en utilisant l'entrypoint `balanceOf()` hérité du contrat `ERC20.sol`.

```solidity
function getContractBalance() public view returns (uint256) {
    return balanceOf(address(this));
}
```

##### Recommandation

Supprimer entièrement la fonction de [la ligne 54 à la ligne 56]. 
Cette suppression permet d'économiser l'exécution d'un OPCODE `JUMP`.

--------

## XXX-L04: Fonction inutile (2)
**Severité:** `Low`
**Target:** `Vesting.sol`

##### Description

La fonction `getContractBalance()` permet de récupérer la balance de tokens ERC-20 détenus dans le contrat. Cependant, il est déjà possible d'effectuer cette action en utilisant l'entrypoint `balanceOf()` du contrat `ERC20.sol` correspondant.

```solidity
function getContractBalance() public view returns (uint256) {
    return vestedToken.balanceOf(address(this));
}
```

##### Recommandation

Supprimer entièrement la fonction de [la ligne 250 à la ligne 252]. 
Cette suppression permet d'économiser l'exécution d'un OPCODE `JUMP`.

--------

## XXX-L05: Variable non utilisée
**Severité:** `Low`
**Target:** `VestingAggregator.sol`

##### Description

Une variable non utilisée est présente dans le code et peut prêter à confusion.

```solidity
address[] public vestingContracts;
```

##### Recommandation

Supprimer la variable inutilisé [`vestingContracts` de la ligne 12](vestingContracts), qui prête à confusion avec la variable `vestingAddresses`.

--------

## XXX-L06: Interface dépréciée
**Severité:** `Low`
**Target:** `IVesting.sol`

##### Description

L'interface `IVesting.sol`, utilisée par le contrat `VestingAggregator.sol`, n'est pas à jour et présente des fonctions dépréciées.

##### Recommandation

Mettre à jour l'interface [`IVesting.sol`].

--------

## XXX-L07: Le code est incorrectement formaté et indenté
**Severité:** `Low`
**Target:** `Entire repository`

##### Description

Le code est incorrectement formaté et indenté sur l'ensemble du projet.

##### Recommandation

Il est primordial de conserver un code correctement formaté et indenté. Il est essentiel de définir une configuration de formatage avec des outils tels que `prettier` ou `foundry` et de l'exécuter systématiquement avant chaque commit pour garantir une excellente qualité du code.

--------

## XXX-M01: La gestion des erreurs n'est pas uniforme
**Severité:** `Medium`
**Target:** `Entire repository`

##### Description

Le code mélange la gestion d'erreur à travers des `require()` et des `custom errors`. Il est impératif de maintenir une cohérence dans la gestion de ces erreurs afin de faciliter la compréhension et la validation des chemins de réussite (happy paths) et d'erreur (bad paths).

##### Recommandation

Il est fortement recommandé de remplacer toutes les erreurs générées par des `require()` par des `custom errors`.

--------

## XXX-M02: Chemins d'erreur non atteints dans la couverture de test
**Severité:** `Medium`
**Target:** `Vesting.sol`

##### Description

Le scénario dans lequel un utilisateur réclame ses tokens via l'entrypoint `claim()` avant l'ouverture du vesting n'a pas été testé dans le projet (voir l'erreur à [la ligne 135]).
```solidity
if (block.timestamp < startDateForCliff + cliffPeriod) {
    // 0% APRES REFLEXION JE SUIS OBLIGE DE LAISSER LA CONDITION
    revert VestingNotStarted();
}
```
Le scénario dans lequel un utilisateur réclame ses tokens via l'entrypoint `claim()`, mais qu'aucun token n'est disponible pour lui, n'a pas été testé dans le projet (voir l'erreur à [la ligne 174]).
```solidity
} else {
    revert ClaimAmountIs0();
}
```


##### Recommandation

Il est fortement recommandé d'ajouter des tests qui valident le fonctionnement de ces scénarios.

--------

## XXX-M03: Chemins non atteints dans la couverture de test
**Severité:** `Medium`
**Target:** `Vesting.sol`

##### Description

Plusieurs scénarios ont été laissés de côté dans la couverture de test.

```solidity
if (totalClaimed > totalVested) {
    metrics[2] = 0;
} else {
    metrics[2] = totalVested - totalClaimed; // Je suppose que c'est la définition du "total libéré"
}

if(block.timestamp < startDateForCliff) {
    metrics[3] = startDateForCliff - block.timestamp; // Temps restant avant TLTC
} else {
    metrics[3] = 0;
}

uint256 vestingEndTime = startDateForCliff + cliffPeriod + totalVestingDuration;
if(block.timestamp < vestingEndTime) {
    metrics[4] = vestingEndTime - block.timestamp; // Temps restant avant TC
} else {
    metrics[4] = 0;
}
```

##### Recommandation

Il est fortement recommandé d'ajouter un test qui valide le fonctionnement de tous ces scénarios manquants.

--------

## XXX-M04: Chemin non atteint dans la couverture de test
**Severité:** `Medium`
**Target:** `VestingAggregator.sol`

##### Description

Un scénario a été laissé de côté dans la couverture de test. Dans le cas où un utilisateur aurait des tokens disponibles suite à l'appel agrégé de la fonction `aggregatedClaim()`, la réclamation auprès du contrat `Vesting.sol` associé n'a pas été testée ([voir ligne 50 à 52]).
```solidity
if (claimableTokens != 0) {
    IVesting(vestingAddresses.at(j)).claim(msg.sender);
}
```

##### Recommandation

Il est fortement recommandé d'ajouter un test qui valide le fonctionnement de ce scénario manquant.

##### Remarque

Il est important de corriger des erreurs similaires dans l'ensemble du code, en plus de ce point soulevé.

--------

## XXX-C01: Claim des tokens possible avant la fin du vesting
**Severité:** `Critical`
**Target:** `Vesting.sol`

##### Description

Lorsque le vesting d'un utilisateur est activé, il a actuellement la possibilité de réclamer la totalité de son vesting avant la date de fin du vesting. En effet, le calcul du vesting ne prend en compte que le montant total du vesting de l'utilisateur, sans considérer les réclamations précédemment effectuées. Par conséquent, un utilisateur malveillant peut effectuer autant d'appels que nécessaire dès le premier jour du vesting pour vider complètement sa position verrouillée.

```solidity
    if (
            startDateForCliff + cliffPeriod + allocation.duration < block.timestamp
        ) {
            // 100 %
            newlyVested += allocation.amount;
        } else {
            // 100000 tokens  - 95% / (nb jours vesting - cliffPeriod) => 0.95 * 100K * 1/700
            uint256 durationVestedSoFar =
                block.timestamp - (startDateForCliff + cliffPeriod);
            /// @dev @audit Critical - A user can claim more tokens than he should.
            /// Since already claimed tokens are not substracted to the claimable amount, 
            /// a user can claim the same tokens ratio multiple times until his total allocation is depleted.
            newlyVested =
                (allocation.amount * durationVestedSoFar) / allocation.duration;
        }

        uint256 tmpTgeAmount = allocation.tgeAmount;

        if ((newlyVested + tmpTgeAmount) > 0) {
            uint256 oldAmount = allocation.amount;
            uint256 newAmount = oldAmount - newlyVested;
            uint256 duration = allocation.duration;

            userAddressToAllocation[_address] = Allocation({
                amount: newAmount,
                duration: duration,
                tgeAmount: 0 // forcément 0
            });

            uint256 claimAmount = newlyVested + tmpTgeAmount;
            totalClaimed += claimAmount;

            vestedToken.safeTransfer(
                _address, claimAmount
            );
        }
```

##### Recommandation

Il est impératif de mettre à jour le calcul des tokens récupérables en prenant en compte soit la dernière date de `claim()` ou en prenant en compte le total de tokens déjà réclamés sur l'ensemble du vesting, puis de soustraire ce montant au calcul actuel.

##### PoC de l'attaque

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "forge-std/Test.sol";
import "../../contracts/Vesting.sol";
import "../../contracts/XXX.sol";

contract VestingPoCTest is Test {
    Vesting internal vestingContract;
    XXX internal XXX;

    struct Allocation {
        uint256 duration;
        uint256 amount;
        uint256 tgeAmount;
    }

    function setUp() public {
        XXX = new XXX();
        vestingContract = new Vesting(address(XXX), 365 days);

        uint256 transferAmount = 500_000_000 * 10e18;
        XXX.transferFromContract(
            address(vestingContract),
            transferAmount
        );

        bytes32 MANAGER_ROLE = vestingContract.MANAGER_ROLE();
        vestingContract.grantRole(MANAGER_ROLE, address(this));
    }

    function test_ClaimingMultipleTimesAtOnce_scenario() public {
        address vester = makeAddr("vester");
        uint256 amount = 100_000 * 10e18;
        uint256 tgeAmount = (amount * 5) / 100; // 5% of amount

        vestingContract.startVesting();
        vestingContract.setAllocationToUser(
            vester,
            amount,
            100 days,
            tgeAmount
        );
        /// `cliffPeriod` is 90 days, so the user can't claim before 90 days.
        /// Vesting duration is 100 days, so the user can claim 1% of vesting each day.

        /// @dev Should not be able to claim before cliff period.
        vm.warp(block.timestamp + 89 days);
        vm.expectRevert();
        vestingContract.claim(vester);

        /// @dev The first day, the caller should get 0 tokens from his vesting,
        /// But he should get his TGE amount.
        vm.warp(block.timestamp + 1 days);
        vestingContract.claim(vester);
        /// @dev Claiming again should revert, since the user already claimed his TGE amount.
        vm.expectRevert();
        vestingContract.claim(vester);

        console.log("Vester balance before claiming vesting is: %s", XXX.balanceOf(vester));

        vm.warp(block.timestamp + 10 days);
        (, uint256 leftAmount, ) = vestingContract.userAddressToAllocation(
            vester
        );
        /// @dev In theory, the user should only claim 10% of his vesting at this point.
        /// However, as long as the user has tokens left in his vesting, he can claim them, 10% at a time.
        while (leftAmount > 10) {
            vestingContract.claim(vester);
            (, leftAmount, ) = vestingContract.userAddressToAllocation(vester);
        }
        console.log("Vester balance after spamming vesting claim is: %s", XXX.balanceOf(vester));
        console.log("Tokens left to claim is: %s", leftAmount);
    }
}
```

--------

## XXX-Q01: Quelle stratégie sera adoptée si un vesting a été mal configuré ?
**Severité:** `Question`
**Target:** `Vesting.sol`

##### Description

Lorsqu'un administrateur configure le vesting d'un utilisateur via [l'entrypoint `setAllocationToUser()`], il n'est actuellement pas possible de le modifier ultérieurement. Est-ce que ce mécanisme sera maintenu pour la suite du projet, ou serait-il préférable d'ajouter un setter par précaution ?

--------

## XXX-Q02: Pourquoi un `totalVestingDuration` est mis en place dans le contrat ?
**Severité:** `Question`
**Target:** `Vesting.sol`

##### Description

Lorsqu'un administrateur configure le vesting d'un utilisateur via [l'entrypoint `setAllocationToUser()`],  il détermine la durée du vesting de l'utilisateur en utilisant la variable `_duration`.
Cependant, se pose la question de savoir si cette configuration devrait être maintenue, étant donné que la variable `totalVestingDuration` définie au niveau global du contrat est disponible et n'est pas utilisée ?
