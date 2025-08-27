# storage-go-starter-kit

Repository Branches
1. Master Branch (Current)
REST API implementation using Gin framework with Swagger documentation.

git checkout master
Features:
RESTful endpoints for upload/download
Swagger UI for API testing
2. CLI Version Branch
Command-line interface implementation available in the cli-version branch.

git checkout cli-version
Features:
Direct file upload/download via CLI
Command-line flags for configuration
SDK Implementation (Master Branch)
Storage Client Setup
// Initialize storage client with network configuration
type StorageClient struct {
    web3Client    *blockchain.Web3Client    // For blockchain transactions
    indexerClient *indexer.Client          // For node management
    ctx           context.Context
}

// Create a new storage client instance
func NewStorageClient(ctx context.Context, privateKey string, useTurbo bool) (*StorageClient, error) {
    // Initialize Web3 client for blockchain interactions
    web3Client := blockchain.MustNewWeb3(EvmRPC, privateKey)

    // Select appropriate indexer based on performance needs
    indexerRPC := IndexerRPCStandard
    if useTurbo {
        indexerRPC = IndexerRPCTurbo
    }

    // Create indexer client for node selection
    indexerClient, err := indexer.NewClient(indexerRPC)
    if err != nil {
        web3Client.Close()
        return nil, fmt.Errorf("failed to create indexer client: %v", err)
    }

    return &StorageClient{
        web3Client:    web3Client,
        indexerClient: indexerClient,
        ctx:           ctx,
    }, nil
}
Understanding Upload & Download
Upload Implementation
The upload process involves both API handling and SDK operations. Here's how it works:

API Endpoint Handler:
func (s *Server) handleUpload(c *gin.Context) {
    // Step 1: Receive and save uploaded file temporarily
    file, _ := c.FormFile("file")
    tempFile := filepath.Join(os.TempDir(), file.Filename)
    c.SaveUploadedFile(file, tempFile)
    defer os.Remove(tempFile)  // Cleanup after processing

    // Step 2: Upload to 0G Storage network
    txHash, rootHash, _ := s.client.UploadFile(tempFile)
    
    // Step 3: Return identifiers to client
    c.JSON(http.StatusOK, UploadResponse{
        RootHash: rootHash,  // Used for later retrieval
        TxHash:   txHash,    // Blockchain transaction reference
    })
}
Internal Upload Process:
func (c *StorageClient) UploadFile(filePath string) (string, string, error) {
    // Step 1: Select storage nodes from network
    nodes, err := c.indexerClient.SelectNodes(c.ctx, 1, DefaultReplicas, nil)
    if err != nil {
        return "", "", fmt.Errorf("failed to select storage nodes: %v", err)
    }
    
    // Step 2: Initialize uploader with nodes
    uploader, err := transfer.NewUploader(c.ctx, c.web3Client, nodes)
    if err != nil {
        return "", "", fmt.Errorf("failed to create uploader: %v", err)
    }
    
    // Step 3: Set timeout and upload file
    ctx, cancel := context.WithTimeout(c.ctx, 5*time.Minute)
    defer cancel()
    
    // Step 4: Execute upload and return identifiers
    txHash, rootHash, err := uploader.UploadFile(ctx, filePath)
    return txHash.String(), rootHash.String(), nil
}
What happens during upload:

File is received via multipart form upload
SDK selects available storage nodes
File is processed into chunks and a Merkle tree is created
Root hash are returned
Blockchain transaction is created and signed
Chunks are uploaded in parallel to storage nodes
Download Implementation
The download process retrieves files using their root hash. Here's the flow:

API Endpoint Handler:
func (s *Server) handleDownload(c *gin.Context) {
    // Step 1: Get file identifier from URL
    rootHash := c.Param("root_hash")
    tempFile := filepath.Join(os.TempDir(), rootHash)
    defer os.Remove(tempFile)  // Cleanup after serving

    // Step 2: Download from 0G Storage network
    err := s.client.DownloadFile(rootHash, tempFile)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    // Step 3: Stream file to client
    c.File(tempFile)
}
Internal Download Process:
func (c *StorageClient) DownloadFile(rootHash, outputPath string) error {
    // Step 1: Find nodes storing the file
    nodes, err := c.indexerClient.SelectNodes(c.ctx, 1, DefaultReplicas, nil)
    if err != nil {
        return fmt.Errorf("failed to select storage nodes: %v", err)
    }
    
    // Step 2: Create downloader instance
    downloader, err := transfer.NewDownloader(nodes)
    if err != nil {
        return fmt.Errorf("failed to create downloader: %v", err)
    }
    
    // Step 3: Ensure output directory exists
    if err := os.MkdirAll(filepath.Dir(outputPath), 0755); err != nil {
        return fmt.Errorf("failed to create output directory: %v", err)
    }
    
    // Step 4: Download with timeout and verification
    ctx, cancel := context.WithTimeout(c.ctx, 5*time.Minute)
    defer cancel()
    
    return downloader.Download(ctx, rootHash, outputPath, true)
}
What happens during download:

Root hash is used to locate the file in the network
SDK queries nodes storing the file
File chunks are downloaded in parallel
Each chunk is verified against the Merkle tree
File is streamed to the client
Complete file is assembled and verified
Usage
Clone the repository:
git clone https://github.com/0glabs/0g-storage-go-starter-kit
Navigate to the project directory:
cd 0g-storage-go-starter-kit
Copy the .env.example file to .env and set your private key:
cp .env.example .env
Start the server:
go run main.go
Access Swagger UI: http://localhost:8080/swagger/index.html

Available Endpoints:

POST /api/v1/upload - Upload a file
Request: multipart/form-data with 'file' field
Response: JSON with root_hash and tx_hash
GET /api/v1/download/{root_hash} - Download a file
Request: root_hash in URL path
Response: File content stream
Network Configuration
const (
    EvmRPC             = "https://evmrpc-testnet.0g.ai"
    IndexerRPCStandard = "https://indexer-storage-testnet-standard.0g.ai"
    IndexerRPCTurbo    = "https://indexer-storage-testnet-turbo.0g.ai"
    DefaultReplicas    = 1 // 1 is the minimum number of replicas
)
Best Practices
Resource Management:

Always close web3Client using defer
Use context with timeout for operations
Clean up temporary files
Error Handling:

Check node selection errors
Validate file existence before upload
Handle network timeouts appropriately
Performance:

Use Turbo nodes for faster operations when needed
Configure appropriate replica count
Implement retry logic for failed operations
Next Steps
Explore advanced SDK features in the 0G Storage Client documentation. Learn more about the 0G Storage Network.
