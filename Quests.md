#### Chapter 1 Day 1 - More on Capabilities

## Quests

Consider a scenario where we have two contracts:
1) An NFT Contract representing music records and a collection to store those records
2) An artist's profile that has a link to the artist's music collection

Contract #1:

```cadence
import NonFungibleToken from 0x03

pub contract Record: NonFungibleToken {

    pub var totalSupply: UInt64

    pub event ContractInitialized()
    pub event Withdraw(id: UInt64, from: Address?)
    pub event Deposit(id: UInt64, to: Address?)

    pub let CollectionStoragePath: StoragePath
    pub let CollectionPublicPath: PublicPath
    pub let MinterStoragePath: StoragePath

    pub resource NFT: NonFungibleToken.INFT {
        pub let id: UInt64
        pub let songName: String

        init(songName: String) {
            self.id = self.uuid
            self.songName = songName
            ExampleNFT.totalSupply = ExampleNFT.totalSupply + 1
        }
    }

    pub resource interface CollectionPublic {
        pub fun deposit(token: @NonFungibleToken.NFT)
        pub fun getIDs(): [UInt64]
        pub fun borrowNFT(id: UInt64): &NonFungibleToken.NFT
        pub fun borrowRecordNFT(id: UInt64): &Record.NFT? {
            post {
                (result == nil) || (result?.id == id):
                    "Cannot borrow the reference: the ID of the returned reference is incorrect"
            }
        }
    }

    pub resource Collection: CollectionPublic, NonFungibleToken.Provider, NonFungibleToken.Receiver, NonFungibleToken.CollectionPublic {
        pub var ownedNFTs: @{UInt64: NonFungibleToken.NFT}

        pub fun withdraw(withdrawID: UInt64): @NonFungibleToken.NFT {
            let token <- self.ownedNFTs.remove(key: withdrawID) ?? panic("missing NFT")
            emit Withdraw(id: token.id, from: self.owner?.address)
            return <- token
        }

        pub fun deposit(token: @NonFungibleToken.NFT) {
            let token <- token as! @Record.NFT
            emit Deposit(id: id, to: self.owner?.address)
            self.ownedNFTs[id] <-! token
        }

        pub fun getIDs(): [UInt64] {
            return self.ownedNFTs.keys
        }

        pub fun borrowNFT(id: UInt64): &NonFungibleToken.NFT {
            return (&self.ownedNFTs[id] as &NonFungibleToken.NFT?)!
        }

        pub fun borrowRecordNFT(id: UInt64): &Record.NFT? {
            if self.ownedNFTs[id] != nil {
                let ref = (&self.ownedNFTs[id] as auth &NonFungibleToken.NFT?)!
                return ref as! &Record.NFT
            }

            return nil
        }

        init () {
            self.ownedNFTs <- {}
        }

        destroy() {
            destroy self.ownedNFTs
        }
    }

    pub fun createEmptyCollection(): @NonFungibleToken.Collection {
        return <- create Collection()
    }

    pub fun createRecord(songName: String): @Record {
      return <- create Record(songName: songName)
    }

    init() {
        self.totalSupply = 0

        self.CollectionStoragePath = /storage/RecordCollection
        self.CollectionPublicPath = /public/RecordCollection

        emit ContractInitialized()
    }
}
```

Contract #2:

```cadence
import Record from 0x01

pub contract Artist {

  pub resource Profile {
    pub let id: UInt64
    pub let name: String
    pub let recordCollection: Capability<&Record.Collection{Record.CollectionPublic}>

    init(name: String, recordCollection: Capability<&Record.Collection{Record.CollectionPublic}>) {
      self.id = self.uuid
      self.name = name
      self.recordCollection = recordCollection
    }
  }

  pub fun createProfile(name: String, recordCollection: Capability<&Record.Collection{Record.CollectionPublic}>): @Profile {
    return <- create Profile(name: name, recordCollection: recordCollection)
  }

}
```

Then, do the following:

1) Write a transaction to save a `@Record.Collection` to the signer's account, making sure to link the appropriate interfaces to the public path.

2) Write a transaction to mint some `@Record.NFT`s to the user's `@Record.Collection`

3) Write a script to return an array of all the user's `&Record.NFT?` in their `@Record.Collection`

4) Write a transaction to save a `@Artist.Profile` to the signer's account, making sure to link it to the public so we can read it

5) Write a script to fetch a user's `&Artist.Profile`, borrow their `recordCollection`, and return an array of all the user's `&Record.NFT?` in their `@Record.Collection` from the `recordCollection`

6) Write a transaction to `unlink` a user's `@Record.Collection` from the public path

7) Explain why the `recordCollection` inside the user's `@Artist.Profile` is now invalid

8) Write a script that proves why your answer to #7 is true by trying to borrow a user's `recordCollection` from their `&Artist.Profile`

## Answer
1. setup account

``` cadence
import Record from 0x01
import NonFungibleToken from 0x03

transaction() {
	prepare(account: AuthAccount) {
		let ref = account.borrow<&Record.Collection>(from: Record.CollectionStoragePath)

		if ref == nil {
			account.save(<- Record.createEmptyCollection() , to: Record.CollectionStoragePath)
			account.link<&Record.Collection{Record.CollectionPublic, NonFungibleToken.Receiver, NonFungibleToken.CollectionPublic}>(Record.CollectionPublicPath, target: Record.CollectionStoragePath)
		}
	}
}
```

2. mint some records
``` cadence
import Record from 0x01
import NonFungibleToken from 0x03

transaction(songName: String) {

	let receiver : &Record.Collection{Record.CollectionPublic}

	prepare(account: AuthAccount) {
		let ref = account.borrow<&Record.Collection>(from: Record.CollectionStoragePath)

		if ref == nil {
			account.save(<- Record.createEmptyCollection() , to: Record.CollectionStoragePath)
			account.link<&Record.Collection{Record.CollectionPublic, NonFungibleToken.Receiver, NonFungibleToken.CollectionPublic}>(Record.CollectionPublicPath, target: Record.CollectionStoragePath)
		}

		self.receiver = account.borrow<&Record.Collection>(from: Record.CollectionStoragePath)!
	}

	execute{
		let nft <- Record.createRecord(songName: songName)
		self.receiver.deposit(token: <- nft)
	}
}
```

3. a script to return an array of all records in the user collection

```cadence
import Record from 0x01

pub fun main(user: Address) : [&Record.NFT]? {
	let cap = getAccount(user).getCapability<&Record.Collection{Record.CollectionPublic}>(Record.CollectionPublicPath)

	if !cap.check() {
		return nil
	}

	let ref = cap.borrow()!
	let array : [&Record.NFT] = []
	for id in ref.getIDs() {
		if let nft = ref.borrowRecordNFT(id: id) {
			array.append(nft)
		}
	}
	return array

}
```

4. store and link artist

``` cadence
import Record from 0x01
import Artist from 0x02

transaction(name: String) {
	prepare(account: AuthAccount) {
		let ref = account.borrow<&Artist.Profile>(from: /storage/Artist)

		let cap = account.getCapability<&Record.Collection{Record.CollectionPublic}>(Record.CollectionPublicPath)

		if ref == nil {
			account.save(<- Artist.createProfile(name: name , recordCollection: cap) , to: /storage/Artist)
			account.link<&Artist.Profile>(/public/Artist, target: /storage/Artist)
		}
	}
}
```

5. fetch capability from artist and return array of nfts
```cadence
import Record from 0x01
import Artist from 0x02

pub fun main(user: Address) : [&Record.NFT]? {

	let profile = getAccount(user).getCapability<&Artist.Profile>(/public/Artist)

	if !profile.check() {
		return nil
	}

	let cap = profile.borrow()!.recordCollection

	if !cap.check() {
		return nil
	}

	let ref = cap.borrow()!
	let array : [&Record.NFT] = []
	for id in ref.getIDs() {
		if let nft = ref.borrowRecordNFT(id: id) {
			array.append(nft)
		}
	}
	return array

}
```

6. unlink Record.Collection from public path

``` cadence
import Record from 0x01

transaction() {

	prepare(account: AuthAccount) {

		account.unlink(Record.CollectionPublicPath,)

	}
}
```

7. Explain why the `recordCollection` inside the user's `@Artist.Profile` is now invalid

Because the capability path is unlinked, borrowing from the capability pointing to the path will return nil (which means the capability is revoked and invalidated)

8.

We should see panic message with `"Cannot borrow recordCollection"` in the script result.

```cadence
import Record from 0x01
import Artist from 0x02

pub fun main(user: Address) : [&Record.NFT] {

	let profile = getAccount(user).getCapability<&Artist.Profile>(/public/Artist)

	if !profile.check() {
		panic("Cannot borrow Profile")
	}

	let cap = profile.borrow()!.recordCollection

	if !cap.check() {
		panic("Cannot borrow recordCollection")
	}

	let ref = cap.borrow()!
	let array : [&Record.NFT] = []
	for id in ref.getIDs() {
		if let nft = ref.borrowRecordNFT(id: id) {
			array.append(nft)
		}
	}
	return array

}
```


# Chapter 1 Day 2 - Private Capabilities

## Quests

1. In the very last transaction of the **Using Private Capabilities** section in today's lesson, there was this line:

```cadence
// Borrow the capability
let minter = &ExampleNFT.NFTMinter = minterCapability.borrow() ?? panic("The capability is no longer valid.")
```

Explain in what scenario this would panic.

2. Explain two reasons why passing around a private capability to a resource you own is different from simply giving users a resource to store in their account.

3. Write (in words) a scenario where you would use private capabilities. It cannot be the same NFT example we saw today.

4. Architect and implement your idea in #3. Show:
- at least one example of you `link`ing a private capability
- at least one example of you storing the private capability inside a resource
- an example of you revoking (`unlink`ing) the capability and showing the capability is now invalid

## Answers
1. When Capability is revoked by the resource owner (i.e. the one who owns NFTMinter), or if the resource is moved by the owner this will panic

2.
	i. Giving users a resource will lose the privileges on managing authorizations. The admin can not stop the resource owner from bad acting because they have no control over the resource and calling the functions in that resource

	ii. Passing a private capability allows people to set up the proxy resource in 2 separate transactions and does not requires co-signing. Which giving resources does.

3. Examples are :
	i. Granting employees' the ability to call admin functions. Which makes sense to revoke their access when they resign.

4. Solution :


```cadence
pub contract Data {

    pub var registry: {UInt64 : UInt64}

	pub let adminStoragePath : StoragePath
	pub let adminPrivatePath : StoragePath
	pub let adminProxyStoragePath : StoragePath
	pub let adminProxyPublicPath : PublicPath

    pub resource interface AdminProxyPublic {
      pub fun ActivateAdmin(_ cap: Capability<&Admin>) {
        pre {
          cap.check(): "This capability is invalid!"
        }
      }
    }

    pub resource AdminProxy: AdminProxyPublic {
      pub var cap: Capability<&Admin>?

      pub fun ActivateAdmin(_ cap: Capability<&Admin>) {
        self.cap = cap
      }

      init() {
        self.minter = nil
      }
    }

    pub fun createProxy(): @AdminProxy {
      return <- create AdminProxy()
    }

    pub resource Admin {
      pub fun updateRegistry(registry) {
        Data.Registry = registry
      }
    }

    init() {
		self.adminStoragePath = /storage/admin
		self.adminPrivatePath = /private/admin

		self.adminProxyStoragePath = /storage/adminProxy
		self.adminProxyPublicPath = /public/adminProxy

        self.account.save(<- create Admin(), to: self.adminStoragePath)
    }
}
```

```cadence
import Data from 0x01

transaction() {
  prepare(signer: AuthAccount) {
    signer.save(<- Data.createProxy(), to: Data.adminProxyStoragePath)

    signer.link<&Data.AdminProxy{Data.AdminProxyPublic}>(Data.adminProxyPublicPath, target: Data.adminProxyStoragePath)
  }
}
```


```cadence
import Data from 0x01

transaction(adminProxyAddress: Address) {
  prepare(signer: AuthAccount) {
    signer.link<&Data.Admin>(Data.adminPrivatePath, target:  Data.adminStoragePath)

    let admin: = signer.getCapability<&Data.Admin>(Data.adminPrivatePath)

    let adminProxy = getAccount(adminProxyAddress).getCapability(Data.adminProxyPublicPath)
              .borrow<&Data.AdminProxy{Data.AdminProxyPublic}>()
              ?? panic("This account does not have a public AdminProxy.")

    adminProxy.activateAdmin(admin)
  }
}
```



```cadence
import Data from 0x01

transaction(registry: {UInt64 : UInt64}) {
  prepare(signer: AuthAccount) {
    let proxy: &Data.AdminProxy = signer.borrow<&Data.AdminProxy>(from: Data.adminProxyStoragePath)!
    let adminCapability: Capability<&Data.Admin> = adminProxy.cap ?? panic("The capability has not been fulfilled.")
    let minter = adminCapability.borrow() ?? panic("The capability is no longer valid.")

	minter.updateRegistry(registry)

  }
}
```

```cadence
import Data from 0x01

transaction(adminProxyAddress: Address) {
  prepare(signer: AuthAccount) {
    signer.unlink(Data.adminPrivatePath)
  }
}
```
