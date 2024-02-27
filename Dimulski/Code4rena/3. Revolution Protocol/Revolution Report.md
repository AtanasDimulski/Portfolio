# [Revolution Protocol Report](https://code4rena.com/reports/2023-12-revolutionprotocol)

## Findings by Dimulski
| Severity | Title | 
|:--:|:--:|
| [M-01](#M-01) | A malicious creator can manipulate QuorumVotes in createPiece  | 

## <a id='M-01'></a>[M-01] A malicious creator can manipulate QuorumVotes in createPiece 
## Lines of code
https://github.com/code-423n4/2023-12-revolutionprotocol/blob/main/packages/revolution/src/CultureIndex.sol#L226-L229

## Vulnerability details
## Impact
When createPiece() in CultureIndex.sol is called it takes the totalSupply ot tokens in the block

```solidity
newPiece.totalVotesSupply = _calculateVoteWeight(
            erc20VotingToken.totalSupply(),
            erc721VotingToken.totalSupply()
        );
```
and calculates the totalVoteSupply by calling the _calculateVoteWeight() function. Later on the totalVoteSupply is used to calculate the quroumVotes needed for the piece to be able to be auctioned newPiece.quorumVotes = (quorumVotesBPS * newPiece.totalVotesSupply) / 10_000; . A malicious creator can back run the createPice transaction and mint himself NontrasferableERC20Votes in the same block the piece is created. Thus the newly minted votes won't count towards the quorumVotes ratio, but he will still be able to vote for that piece because he minted tokens in the same block which the piece was created. For example if at the time of piece creation the tokens supply is 10 and quorumVotesBPS is 60%, that means for the piece to be eligible for auction is should have at least 6 token votes. Now if a malicious piece creator back runs the transaction and mint 6 tokens he can guarantee that his piece will successfully pass the quorumVotes requirement. However his 6 mitned tokens after a piece is created shouldn't be eligible for voting.As well as the correct quorum should be 60% * 16 = 3.75(I have used small integers to better ilustrate the problem, 3.75 wont round down as we are dealing with 1e18 integers, also these are just example values). This can be abused by malicious creators especially in the beginning as they are less total votes, if he is successful in passing the quorum in such a way and his piece is the top voted he will receive the rewards associated with a piece being successfully auctioned. This also breaks an Invarian specified by the protocol : Only snapshotted (at art piece creation block) vote weights should be able to update the total vote weight of the art piece. eg: If you received votes after snapshot date on the art piece, you should have 0 votes. Keep in mind that transactions are ordered firstly by nonce in the MEV pool, so back running a transaction is not that hard when you first create a createPiece() transaction followed by calling buyToken() in the ERC20TokenEmitter.sol contract.

## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Instead of

```soldiity
newPiece.totalVotesSupply = _calculateVoteWeight(
            erc20VotingToken.totalSupply(),
            erc721VotingToken.totalSupply()
        );
```

use

```solidity
newPiece.totalVotesSupply = _calculateVoteWeight(  
    ec20VotingToken.getPastTotalSupply(block.number -1),
    erc721VotingToken.getPastTotalSupply(block.number -1)
);
```
## Assessed type
Timing
