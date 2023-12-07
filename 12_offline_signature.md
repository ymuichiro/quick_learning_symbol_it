# 12.Firme a freddo (offline)

Nel capitolo sui `Lock`, sono state descritte le transazioni con lock il cui parametro è un determinato valore hash, e le transazioni di gruppo `Aggregate transaction`, la cui esecuzione richiede firme multiple (`online signatures`).  
Questo capitolo descrive come firmare a freddo (`offline`), che consiste nel raccogliere le firme in anticipo e poi propagare la transazione ai nodi della rete blockchain.

## Procedura

Alice crea e firma una transazione, quindi Bob ne appone la firma e la restituisce ad Alice. Ora Alice è pronta a combinare le due transazioni e propagarle alla rete in un colpo solo.

## 12.1 Creazione della Transazione

```js
bob = sym.Account.generateNewAccount(networkType);

innerTx1 = sym.TransferTransaction.create(
  undefined,
  bob.address,
  [],
  sym.PlainMessage.create("tx1"),
  networkType
);

innerTx2 = sym.TransferTransaction.create(
  undefined,
  alice.address,
  [],
  sym.PlainMessage.create("tx2"),
  networkType
);

aggregateTx = sym.AggregateTransaction.createComplete(
  sym.Deadline.create(epochAdjustment),
  [
    innerTx1.toAggregate(alice.publicAccount),
    innerTx2.toAggregate(bob.publicAccount),
  ],
  networkType,
  []
).setMaxFeeForAggregate(100, 1);

signedTx = alice.sign(aggregateTx, generationHash);
signedHash = signedTx.hash;
signedPayload = signedTx.payload;

console.log(signedPayload);
```

###### Output esemplificativo

```js
>580100000000000039A6555133357524A8F4A832E1E596BDBA39297BC94CD1D0728572EE14F66AA71ACF5088DB6F0D1031FF65F2BBA7DA9EE3A8ECF242C2A0FE41B6A00A2EF4B9020E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26000000000198414100AF000000000000D4641CD902000000306771D758886F1529F9B61664B0450ED138B27CC5E3AE579C16D550EDEE5791B00000000000000054000000000000000E5C72B0D5946C1EFEE7E5317C5985F106B739BB0BC07E4F9A288417B3CD6D26000000000198544198A1BE13194C0D18897DD88FE3BC4860B8EEF79C6BC8C8720400000000000000007478310000000054000000000000003C4ADF83264FF73B4EC1DD05B490723A8CFFAE1ABBD4D4190AC4CAC1E6505A5900000000019854419850BF0FD1A45FCEE211B57D0FE2B6421EB81979814F629204000000000000000074783200000000
```

Alice ha firmato e l'output è `signedHash`, `signedPayload`. 
A questo punto passare `signedPayload` a Bob chiedendogli di firmare.

## 12.2 Firma del cointestatario Bob

Per ricostruire la transazione partendo dal `signedPayload` che gli ha comunicato Alice.

```js
tx = sym.TransactionMapping.createFromPayload(signedPayload);
console.log(tx);
```

###### Output esemplificativo

```js
> AggregateTransaction
    cosignatures: []
    deadline: Deadline {adjustedValue: 12197090355}
> innerTransactions: Array(2)
      0: TransferTransaction {type: 16724, networkType: 152, version: 1, deadline: Deadline, maxFee: UInt64, …}
      1: TransferTransaction {type: 16724, networkType: 152, version: 1, deadline: Deadline, maxFee: UInt64, …}
    maxFee: UInt64 {lower: 44800, higher: 0}
    networkType: 152
    payloadSize: undefined
    signature: "4999A8437DA1C339280ED19BE0814965B73D60A1A6AF2F3856F69FBFF9C7123427757247A231EB89BB8844F37AC6F7559F859E2FDE39B8FA58A57F36DDB3B505"
    signer: PublicAccount
      address: Address {address: 'TBXUTAX6O6EUVPB6X7OBNX6UUXBMPPAFX7KE5TQ', networkType: 152}
      publicKey: "D4933FC1E4C56F9DF9314E9E0533173E1AB727BDB2A04B59F048124E93BEFBD2"
    transactionInfo: undefined
    type: 16705
    version: 1
```

Assicurarsi che il corpo della transazione `payload` sia stato già firmato da Alice.

```js
Buffer = require("/node_modules/buffer").Buffer;
res = tx.signer.verifySignature(
  tx.getSigningBytes(
    [...Buffer.from(signedPayload, "hex")],
    [...Buffer.from(generationHash, "hex")]
  ),
  tx.signature
);
console.log(res);
```

###### Output esemplificativo

```js
> true
```

Una volta verificato che il payload è stato effettivamente firmato da Alice, Bob potrà a sua volta apporre la propria firma di cofirmatario.

```js
bobSignedTx = sym.CosignatureTransaction.signTransactionPayload(
  bob,
  signedPayload,
  generationHash
);
bobSignedTxSignature = bobSignedTx.signature;
bobSignedTxSignerPublicKey = bobSignedTx.signerPublicKey;
```
Dopo aver apposto la firma Bob comunica `bobSignedTxSignature` e `bobSignedTxSignerPublicKey` ad Alice.

Se Bob potesse creare tutte le firme necessarie, allora potrebbe anche eseguire la propagazione senza dover comunicare più nulla ad Alice.

## 12.3 Propagazione fatta da Alice

Dopo aver ricevuto `bobSignedTxSignature` e `bobSignedTxSignerPublicKey` da Bob, Alice prepara il payload della transazione che li contiene e firmando in anticipo a proprio nome.

```js
signedHash = sym.Transaction.createTransactionHash(
  signedPayload,
  Buffer.from(generationHash, "hex")
);
cosignSignedTxs = [
  new sym.CosignatureSignedTransaction(
    signedHash,
    bobSignedTxSignature,
    bobSignedTxSignerPublicKey
  ),
];

recreatedTx = sym.TransactionMapping.createFromPayload(signedPayload);

cosignSignedTxs.forEach((cosignedTx) => {
  signedPayload +=
    cosignedTx.version.toHex() +
    cosignedTx.signerPublicKey +
    cosignedTx.signature;
});

size = `00000000${(signedPayload.length / 2).toString(16)}`;
formatedSize = size.substr(size.length - 8, size.length);
littleEndianSize =
  formatedSize.substr(6, 2) +
  formatedSize.substr(4, 2) +
  formatedSize.substr(2, 2) +
  formatedSize.substr(0, 2);

signedPayload =
  littleEndianSize + signedPayload.substr(8, signedPayload.length - 8);
signedTx = new sym.SignedTransaction(
  signedPayload,
  signedHash,
  alice.publicKey,
  recreatedTx.type,
  recreatedTx.networkType
);

await txRepo.announce(signedTx).toPromise();
```

L'ultima parte, nel ciclo che consiste nell'aggiungere una serie di firme è più complessa perchè modifica direttamente il payload cambiandone la dimensione.
Se la chiave privata di Alice potesse essere usata per firmare ancora la transazione, si potrebbero generare transazioni cointestate `cosignSignedTxs` nel modo seguente.

```js
resignedTx = recreatedTx.signTransactionGivenSignatures(
  alice,
  cosignSignedTxs,
  generationHash
);
await txRepo.announce(resignedTx).toPromise();
```

## 12.4 Consigli pratici

### Oltre il mercato di scambio

Diversamente dalle transazioni di tipo `bonded` (legate), non è necessario pagare
commissioni (10 XYM), per gli `hashlock`.
Essendo il payload condivisibile con altri, un venditore potrà creare preventivamente tanti payload quanti i possibili acquirenti potenziali ed attendere che partano le negoziazioni.
(Andrebbero applicati dei meccanismi di esclusione. Per esempio, includendo una ricevuta NFT completa nella transazione di gruppo, per impedire che più transazioni vengano eseguite separatamente).
Non è necessario creare un mercato di scambio dedicato per queste negoziazioni.
Gli utenti possono usare una politica di tipo social network oppure sviluppare un'istanza di mercato dedicato istantanea.
Fare attenzione in particolare al rischio di intercettazione maligna, degli hash delle richieste di firma. Essendo questi scambiati in modalità offline, è necessario generare e firmare i payload da una fonte verificabile).
