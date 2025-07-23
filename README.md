# Akave SDK CLI

The **Akave SDK CLI** (`akavesdk`) is a command-line tool designed to streamline interactions with Akave's decentralized storage platform. With this SDK, developers can efficiently manage buckets, upload and download files, and interact with metadata across Akave nodes. It supports both standard API operations and an advanced streaming API for high-performance file management, as well as an IPC API for blockchain-based data operations.

Whether you're building a new integration or managing data across nodes, this SDK provides robust capabilities to help you achieve seamless, scalable storage solutions.

```Base commit: tag v0.2.0```.

## Build and test instructions
Requirements: Go 1.23+

`make build` - outputs a cli binary into `bin/akavecli`.<br>
`make test` - runs tests.<br>
Look at `Makefile` for details.

# Akave Node API
The Akave Node API provides a set of gRPC services for interacting with the Akave node. Below is a description of the model and available functions.

### Metadata model
#### Bucket model
- unique identifier for a bucket
    ```go
    type ID [32]byte
    ```
- bucket metadata
    ```go
    type Bucket struct {
      ID        ID
      Name      string
      CreatedAt time.Time
    }
    ```

#### File model
- content identifier for files, chunks and blocks
    ```go
    type CID string
    ```
- unique key to identify a file in the store
    ```go
    type FileID struct {
      BucketID buckets.ID
      Name     string
    }
    ```
- file metadata of a streaming file
    ```go
    type FileMeta struct {
      FileID

      StreamID    uuid.UUID
      RootCID     CID
      Size        int64
      CreatedAt   time.Time
      CommittedAt time.Time
    }
    ```
- streaming file model: metadata + chunks it is made of
    ```go
    type FileV2 struct {
      FileMeta

      Chunks []ChunkMeta
    }
    ```
- chunk metadata
    ```go
    type ChunkMeta struct {
      CID  CID
      Size int64
    }
    ```
- chunk metadata with blocks
    ```go
    type Chunk struct {
      ChunkMeta
      Blocks []Block
    }
    ```
- block metadata
    ```go
    type Block struct {
      CID  CID
      Size int64
    }
    ```

### Bucket API

| Endpoint       | Description                                                                                                                                         |
|----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| `BucketCreate` | Creates a new bucket. The request sent on a node creates a bucket on this node and shares the fact of creation with all other nodes in the network. |
| `BucketView`   | Retrieves details of a specific bucket.                                                                                                             |
| `BucketList`   | Lists all buckets in the network.                                                                                                                   |
|| `BucketDelete` | Deletes a specific bucket. Fact of deletion is shared among all nodes in network. For now, only *soft delete* is implemented. Soft delete marks the bucket as deleted but data may remain until cleanup cycles remove it. |

### Streaming File API

In streaming API file is split into chunks of 32 MB max size where each chunk contains blocks with data(max size of a block is 1 MB).
SDK, when reading the file, attempts to read at most 32 MB as 1 read operation from the source, then builds a DAG for chunk where root CID of a DAG is chunk CID.
Then SDK creates an upload receipt of the chunk using `FileUploadChunkCreate` api endpoint. Receipt describes where each block should be uploaded to,
as upload of blocks is performed simultaneously on multiple akave nodes. Once the chunk is fully uploaded, the process repeats for other chunks, until the file is fully read.
After that SDK performs "commit" operation(using `FileUploadCommit` endpoint), signaling akave nodes, that upload of given file has finished and passing file root CID.
File root CID is calculated incrementally using chunk CIDs.


| Endpoint                  | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `FileUploadCreate`        | Initiates a file upload. Creates a unique stream-id for the file and stores it together with file data (e.g. fileName, bucket, creation_date). Shares the fact of such storing among all nodes in the network. This unique stream-id is used by the client in further requests.                                                                                                                                                                                                                                            |
| `FileUploadChunkCreate`   | Stores the given chunk (cid, size and blocks metadata) and returns the receipt where each block should be uploaded to. Also shares the fact of storing the chunk among all nodes in the network.                                                                                                                                                                                                                                                                                                                           |
| `FileUploadBlock`         | Uploads the given block (block's data) via grpc streaming to the node address returned in response to **FileUploadChunkCreate**. If the replication is enabled on a node, the node also replicates this block to some other nodes selected randomly (**replication_factor** defines to how many nodes a block should be replicated to). Also node stores information about peer ID of a node which now has this block (current node and replicated nodes) and shares this information with all other nodes in the network. |
| `FileUploadCommit`        | Signals that upload operation is completed providing akave node with file's **root_cid**. Before this operation the file is "invisible": you can't get info about it or download it. After this operation you can't upload more blocks or chunks to this file.                                                                                                                                                                                                                                                             |
| `FileDownloadCreate`      | Initiates file download. Fetches the file's metadata and its breakdown on chunks: list of chunks this file is made of.                                                                                                                                                                                                                                                                                                                                                                                                     |
| `FileDownloadChunkCreate` | Creates the "download receipt" for a chunk: list of blocks this chunk is made of and from where each block can be downloaded from.                                                                                                                                                                                                                                                                                                                                                                                         |
| `FileDownloadBlock`       | Downloads the block via grpc streaming from a node which address is taken from the response to **FileDownloadChunkCreate**.                                                                                                                                                                                                                                                                                                                                                                                                |
| `FileList`                | Lists all the files in the given bucket. Only file metadata is returned.                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `FileView`                | Fetches the metadata of one particular file.                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
|| `FileDelete`              | "Soft" deletes the file in a block. The node also shares the information about this operation among all nodes in a network. Soft delete marks the file as deleted but data may remain until cleanup cycles remove it.                                                                                                                                                                                                                                               |

> NOTE: Sharing among nodes functionality uses libp2p pubsub. "Soft" deletes means marking an object with delete flag in db table, while "hard" deletes permanently remove the data and associated blocks.

### Akave Node IPC API

The Akave Node IPC API provides access to gRPC service that interacts with Akave node, and IPC deployed smart-contract that operates with metadata.
Uses same File, Bucket, Block and Chunk models as regular Akave Node API


| Endpoint             | Description                                                                                                                                                                                                                                                                                                                                                                                                                              |
|----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ConnectionParams`   | Retrieves dial URI and deployed smart contract address to interract with.                                                                                                                                                                                                                                                                                                                                                                |
| `BucketCreate`       | Unimplemented. Functionality calls from SDK side.                                                                                                                                                                                                                                                                                                                                                                                        |
| `BucketView`         | Retrieves single bucket metadata by bucket name and creator address. Calls smart contract method GetBucketByName, transforms response into API bucket model.                                                                                                                                                                                                                                                                             |
| `BucketList`         | Retrieves all buckets metadata (ID, name, created at). For now doesn't sort by creator address.                                                                                                                                                                                                                                                                                                                                          |
|| `BucketDelete`       | Soft deletes a bucket by marking it as deleted. The bucket metadata is preserved but marked as deleted in the smart contract.                                                                                                                                                                                                                                                                                                        |
| `FileView`           | Retrieves single file metadata by file name, bucket name, creator address. Calls smart contract GetBucketByName, to receive it's ID, then GetFileByName and transforms response into API file model.                                                                                                                                                                                                                                     |
| `FileList`           | Retrieves all files of bucket (by name and creator address) metadata. Calls smart contract GetBucketByName, then GetFileByName through all bucket's file id's list.                                                                                                                                                                                                                                                                      |
|| `FileDelete`         | Hard deletes a file and optionally all its blocks. This operation permanently removes file metadata and associated blocks from the smart contract.                                                                                                                                                                                                                                                                                        |
| `FileUploadCreate`   | Initiates a file upload. Selects node for each file block to upload.                                                                                                                                                                                                                                                                                                                                                                     |
| `FileUploadBlock`    | Uploads the given block (block's data) via grpc streaming to the node address. If the replication is enabled on a node, the node also replicates this block to some other nodes selected randomly (**replication_factor** defines to how many nodes a block should be replicated to). Also node stores information about peer ID of a node which now has this block (current node and replicated nodes) and stores it on smart contract. |
| `FileDownloadCreate` | Fetches the file's metadata and its breakdown on blocks: list of blocks this file is made of from smart contract by calling GetBucketByName, GetFileByName, GetFileBlockById respectively. Assigns peer to each block.                                                                                                                                                                                                                   |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `FileDownloadBlock`  | Downloads the block by cid via grpc streaming from a node which address picked in **FileDownloadCreate**.                                                                                                                                                                                                                                                                                                                                |

# IPC Storage smart contract

The Storage smart contract contains collection of methods that provides access to bucket, files, chunks and peerblocks metadata.

#### Bucket model
- bucket metadata
    ```go
    type StorageBucket struct {
        Id        [32]byte
        Name      string
        CreatedAt *big.Int
        Owner     common.Address
        Files     [][32]byte
    }
    ```

#### File model
- file metadata
    ```go
    type StorageFile struct {
        Id        [32]byte
        Cid       []byte
        BucketId  [32]byte
        Name      string
        Size      *big.Int
        CreatedAt *big.Int
        Blocks    [][32]byte
    }
    ```

#### Block model
- block metadata
    ```go
    type StorageFileBlock struct {
        Id     [32]byte
        Cid    []byte
        FileId [32]byte
        Size   *big.Int
    }
    ```
  
#### Functions
- CreateBucket(opts *bind.TransactOpts, name string) (*types.Transaction, error) {...} <br>
creates bucket, requires keyed transactor, string name. Used in IPC SDK CreateBucket.
- DeleteBucket(opts *bind.TransactOpts, id [32]byte, name string) (*types.Transaction, error) {...} <br>
deletes bucket, requires same keyed transactor as creator of this bucket, bucket id 32byte array, string name.
Used in IPC SDK DeleteBucket.
- GetBucketByName(opts *bind.CallOpts, name string) (StorageBucket, error) {...} <br>
retrieves bucket metadata, requires From(address of creator) to be filled in request (f.e. &bind.CallOpts{From: client.Auth.From}), and bucket name.
Used to get bucket's ID in IPC SDK CreateBucket, FileDelete, CreateFileUpload, and in IPC endpoint BucketView, FileView, FileList, FileDownloadCreate.
- AddFile(opts *bind.TransactOpts, cid []byte, bucketId [32]byte, name string, size *big.Int) (*types.Transaction, error) {...} <br>
adds file metadata, requires keyed transactor, content identifier (bytes), bucket id, name and size of *big.Int format, returns transaction.
Used in IPC SDK CreateFileUpload.
- DeleteFile(opts *bind.TransactOpts, id [32]byte, bucketId [32]byte, name string, force bool) (*types.Transaction, error) {...} <br>
deletes file metadata, and if force flag set to true all file blocks. required same keyed transactor as while file add func call, file id, bucket id, name.
Used in IPC SDK FileDelete.
- GetFileByName(opts *bind.CallOpts, bucketId [32]byte, name string) (StorageFile, error) {...} <br>
retrieves file metadata by bucket id, file name.
Used in FileDelete, CreateFileUpload, and IPC endpoint FileDownloadCreate, FileView.
- AddFileBlock(opts *bind.TransactOpts, fileId [32]byte, cid []byte, size *big.Int) (*types.Transaction, error) {...} <br>
adds file block metadata, requires keyed transactor, file id 32byte array, content identifier bytes, size *big.int.
Used in IPC SDK CreateFileUpload.
- GetFileBlockById(opts *bind.CallOpts, id [32]byte) (StorageFileBlock, error) {...} <br>
retrieves file block by id, requires block id as 32byte array.
Used in IPC endpoint FileDownloadCreate and IPC SDK CreateFileDownload.
- AddPeerBlock(opts *bind.TransactOpts, peerId []byte, cid []byte, isReplica bool) (*types.Transaction, error) {...} <br>
adds peer block, requires keyed transactor, peerId (node id) bytes, content identifier bytes, is block replicated bool.
Used in IPC endpoint FileUploadBlock.
- GetPeersByPeerBlockCid(opts *bind.CallOpts, cid []byte) ([][]byte, error) {...} <br>
returns all peerIds that has peer block with given content identifier, requires cid bytes.
Used in IPC endpoint FileDownloadCreate.
<br>

# SDK

### SDK API

| Function Name        | Description                                                                                                                                                   |
|----------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `New`                | Creates a new instance of the SDK client                                                                                                                      |
| `CreateBucket`       | Creates a new bucket with the specified name                                                                                                                  |
| `ViewBucket`         | Retrieves details of a specific bucket by its name                                                                                                            |
| `ListBuckets`        | Lists all buckets available in the network                                                                                                                    |
|| `DeleteBucket` | "Soft" Deletes a specific bucket by its name. Marks the bucket as deleted but data may remain until cleanup processes remove it.                                                                                                                  |
| `StreamingAPI`       | Returns sdk instance that works with streaming file api                                                                                                       |
| `ListFiles`          | Lists all streamed files in a specified bucket.                                                                                                               |
| `FileInfo`           | Retrieves metadata of a specific streamed file by its name and bucket                                                                                         |
| `CreateFileUpload`   | Initiates a file upload to a specified bucket by creating file's stream-id                                                                                    |
| `Upload`             | Uploads file's data to a file identified by stream-id. Splits file on chunks and performs chunk upload to different nodes                                     |
| `CreateFileDownload` | Initiates a file download from a specified bucket. Gets a receipt that describes which chunks the file consists of                                            |
| `Download`           | Using the receipt returned from `CreateFileDownload` endpoint downloads the file sequentially by chunks. Fetches peer block addresses of blocks of each chunk |
|| `FileDelete`         | Soft deletes a specific file by its name and bucket id. Marks the file as deleted but data may remain until cleanup processes remove it.                                                                                                        |

### SDK DAG utilities
- `ChunkDAG` a struct that contains node's(in context of a DAG) metainformation: CID, sizes, block metadata
- `DAGRoot` helps to build file's root CID. On each file chunk you have to add a link to chunk(specifiying chunk's CID and its node sizes)
- `BuildDAG` build `ChunkDAG` from a given file. This DAG is of flat structure, meaning it has 1 root and all blocks are children of this single root

### SKD IPC API

| Function Name | Description                                                                                                                                                                                  |
|---------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `CreateBucket` | Creates a new bucket with the specified name calling IPC contract CreateBucket, checks if it exists, returns bucket id.                                                                      |
| `ViewBucket` | Retrieves details of a specific bucket by its name.                                                                                                                                          |
| `ListBuckets` | Lists all buckets stored in IPC contract.                                                                                                                                                    |
| `DeleteBucket` | Soft deletes a specific bucket by its name.                                                                                                                                                  |
| `IPC` | Returns ipc sdk instance that works ipc contract.                                                                                                                                            |
| `ListFiles` | Lists all files from a specific bucket.                                                                                                                                                      |
| `FileInfo` | Retrieves metadata of a specific file by its name and bucket's name.                                                                                                                         |
| `CreateFileUpload` | Initiates a file upload to a specified bucket, adding file's metadata and it's blocks metadata to IPC smart contract. Then Assigns each file blocks to different nodes calling ipc endpoint. |
| `Upload` | Splits file on blocks and performs chunk upload to different nodes. Add peer-block to smart contract, that stores info about where each file block stored node-wise.                         |
| `CreateFileDownload` | Initiates a file download from a specified bucket. Gets a receipt that describes which chunks the file consists of.                                                                          |
| `Download` | Using the receipt returned from `CreateFileDownload` endpoint downloads the file by blocks, previously fetches peer block addresses of blocks                                                |
|| `FileDelete` | Hard delete (with all blocks) a specific file by its name and bucket's name. Permanently removes the file and all associated blocks from storage.                                                                                                                 |

<br>

# Akave CLI

## Key Features

- **Bucket Management**: Create, view, list, and delete buckets.
- **File Management**: Upload, download, view, list, and delete files.
- **Streaming API**: Interact with the streaming file API for efficient file operations.
- **IPC API**: Interact with the blockchain API for bucket and file operations.

Encryption key in flag `-e`(or `--encryption-key` in long form) for upload and download file commands is optional, without using it data will be unencrypted. Key must be a hex encoded string.
The key length **must be 32 bytes** long.

Disable erasure coding flag `disable-erasure-coding` ensures that file is not erasure encoded before being uploaded to akave node(erasure coding is enabled by default).

## Commands

### Bucket Commands
- **Create Bucket**: Creates a new bucket.
  ```sh
  akavecli bucket create <bucket-name> --node-address=localhost:5000
  ```
- **View Bucket**: Retrieves details of a specific bucket.
  ```sh
  akavecli bucket view <bucket-name> --node-address=localhost:5000
  ```
- **List Buckets**: List all available buckets.
  ```sh
  akavecli bucket list --node-address=localhost:5000
  ```
- **Delete Bucket**: Soft deletes a specific bucket.
  ```sh
  akavecli bucket delete <bucket-name> --node-address=localhost:5000
  ```
  **Note**: Soft delete marks the bucket as deleted but doesn't immediately remove the data. The data may still be recoverable until the next cleanup cycle.

### Streaming API
- **List Files**: Lists all files in a specified bucket.
  ```sh
  akavecli files-streaming list <bucket-name> --node-address=localhost:5000
  ```
- **File Info**: Retrieves metadata of a specific file.
  ```sh
  akavecli files-streaming info <bucket-name> <file-name> --node-address=localhost:5000
  ```
- **Upload File**: Uploads a file to a specified bucket from the local file system(`-e` is optional, key length **must be 32 bytes** long, `disable-erasure-coding` is optional)
  ```sh
  akavecli files-streaming upload <bucket-name> <file-path> -e="encryption-key" --disable-erasure-coding --node-address=localhost:5000
  ```
- **Download File**: Downloads a file from a specified bucket to destination folder(`-e` is optional, key length **must be 32 bytes** long, `disable-erasure-coding` is optional)
  ```sh
  akavecli files-streaming download <bucket-name> <file-name> <destination-folder> -e="encryption-key" --disable-erasure-coding --node-address=localhost:5000
  ```
  <small>`<file-name>` here is the last segment in `<file-path>` of Upload command</small>
  
- **Delete File**: Soft deletes a specific file.
  ```sh
  akavecli files-streaming delete <bucket-name> <file-name> --node-address=localhost:5000
  ```
  **Note**: Soft delete marks the file as deleted but doesn't immediately remove the data. The data may still be recoverable until the next cleanup cycle.
  
  <small>`<file-name>` here is the last segment in `<file-path>` of Upload command</small>

### Akave IPC CLI

## Authentication Methods

### Recommended: Secure Storage Methods

#### Using Wallets (Recommended)
Create and manage wallets securely:
```sh
# Create a new wallet
akavecli wallet create my-wallet

# List available wallets
akavecli wallet list

# Use wallet for operations (default: uses first available wallet)
akavecli ipc bucket create <bucket-name> --node-address=localhost:5000

# Use specific wallet
akavecli ipc bucket create <bucket-name> --node-address=localhost:5000 --account=my-wallet
```

#### Using Environment Variables
Set your private key as an environment variable:
```sh
# Set environment variable
export AKAVE_PRIVATE_KEY="your-private-key-here"

# Use in commands
akavecli ipc bucket create <bucket-name> --node-address=localhost:5000
```

### Alternative: Inline Private Key (Testing Only)
⚠️ **WARNING**: Only use inline private keys for testing. Never use this method in production or with valuable keys.

```sh
akavecli ipc bucket create <bucket-name> --node-address=localhost:5000 --private-key="test-private-key"
```

## Commands

### Bucket Commands
- **Create Bucket**: Creates a new bucket.
  ```sh
  # Using wallet (recommended)
  akavecli ipc bucket create <bucket-name> --node-address=localhost:5000
  
  # Using specific wallet
  akavecli ipc bucket create <bucket-name> --node-address=localhost:5000 --account=my-wallet
  ```
- **Delete Bucket**: Soft deletes a specific bucket.
  ```sh
  # Using wallet (recommended)
  akavecli ipc bucket delete <bucket-name> --node-address=localhost:5000
  ```
  **Note**: Soft delete marks the bucket as deleted but doesn't immediately remove the data. The data may still be recoverable until the next cleanup cycle. For permanent deletion, contact your system administrator.
  
- **View Bucket**: Retrieves details of a specific bucket.
  ```sh
  # Using wallet (recommended)
  akavecli ipc bucket view <bucket-name> --node-address=localhost:5000
  ```
- **List Buckets**: List all available buckets.
  ```sh
  # Using wallet (recommended)
  akavecli ipc bucket list --node-address=localhost:5000
  ```
### File Commands
- **List Files**: List all files in a bucket.
  ```sh
  # Using wallet (recommended)
  akavecli ipc file list <bucket-name> --node-address=localhost:5000
  ```
- **File Info**: Retrieves file information.
  ```sh
  # Using wallet (recommended)
  akavecli ipc file info <bucket-name> <file-name> --node-address=localhost:5000
  ```
- **Upload File**: Uploads a file to a bucket(`-e` is optional, key length **must be 32 bytes** long, `disable-erasure-coding` is optional)
  ```sh
  # Using wallet (recommended)
  akavecli ipc file upload <bucket-name> <file-path> -e="encryption-key" --disable-erasure-coding --node-address=localhost:5000
  ```
- **Download File**: Downloads a file from a bucket(`-e` is optional, key length **must be 32 bytes** long, `disable-erasure-coding` is optional)
  ```sh
  # Using wallet (recommended)
  akavecli ipc file download <bucket-name> <file-name> <destination-path> -e="encryption-key" --disable-erasure-coding --node-address=localhost:5000
  ```
- **Delete File**: Hard deletes a specific file and all its blocks.
  ```sh
  # Using wallet (recommended)
  akavecli ipc file delete <bucket-name> <file-name> --node-address=localhost:5000
  ```
  **Note**: IPC file deletion is a **hard delete** that permanently removes the file and all its associated blocks from storage. This operation cannot be undone.
