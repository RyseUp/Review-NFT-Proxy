# Code Review for NFT Proxy Project

**Candidate**: Nguyen Hoang Minh  
**LinkedIn**: [Nguyen Hoang Minh](https://www.linkedin.com/in/minh-nguyen-9a1729234/)  
**Repository Link**: [NFT Proxy GitHub Repo](https://github.com/alphabatem/nft-proxy.git)

---

## 1. Introduction & Scope

This document provides a static code review for the NFT Proxy project, focusing on identifying bugs, design issues, and opportunities for optimization.

---

## 2. Overview of the Project

The NFT Proxy service:
- Serves NFT data and images from the Solana blockchain.
- Fetches, resizes, and caches images locally.
- Uses SQLite for storing metadata.
- Exposes HTTP endpoints via Gin.

---

## 3. Critical Issues

### 3.1 SQL Injection Risk
- **Location**: [`/cli/reload_hashlist.go`](https://github.com/alphabatem/nft-proxy/blob/main/cli/reload_hashlist.go)
- **Issue**: Direct concatenation of user input in the SQL `WHERE` clause.
- **Why It’s Critical**: An attacker could craft malicious inputs, potentially modifying the SQL query to access or destroy data.

**Code Before**:
```go
query := fmt.Sprintf("mint IN (%s)", strings.Join(hashes, ","))
err = db.Db().Exec(query).Error
```

**Improved Code**:
```go
placeholders := make([]string, len(hashes))
args := make([]interface{}, len(hashes))

for i, h := range hashes {
    placeholders[i] = "?"
    args[i] = h
}
query := "mint IN (" + strings.Join(placeholders, ",") + ")"

err = db.Db().
    Where(query, args...).
    Delete(&nft_proxy.SolanaMedia{}).Error
```

---

### 3.2 Unchecked Errors for Critical Function
- **Location**: [`/service/solana.go`](https://github.com/alphabatem/nft-proxy/blob/main/service/solana.go)
- **Issue**: The `FindTokenMetadataAddress` function returns an error that is currently being discarded.

**Code Before**:
```go
ata, _, _ := svc.FindTokenMetadataAddress(key, solana.TokenMetadataProgramID)
ataT22, _, _ := svc.FindTokenMetadataAddress(key, solana.MustPublicKeyFromBase58("META4s4fSmpkTbZoUsgC1oBnWB31vQcmnN8giPw51Zu"))
```

**Improved Code**:
```go
ata, bump, err := svc.FindTokenMetadataAddress(key, solana.TokenMetadataProgramID)
if err != nil {
    return nil, 0, fmt.Errorf("failed to find token metadata address: %w", err)
}

ataT22, bumpT22, err := svc.FindTokenMetadataAddress(
   key,
   solana.MustPublicKeyFromBase58("META4s4fSmpkTbZoUsgC1oBnWB31vQcmnN8giPw51Zu"),
)
if err != nil {
   return nil, 0, fmt.Errorf("failed to find token metadata address for T22: %w", err)
}
```

---

### 3.3 No Graceful Shutdown Handling
- **Location**: [`/runtime/main.go`](https://github.com/alphabatem/nft-proxy/blob/main/runtime/main.go)
- **Issue**: The application does not handle `SIGINT` or `SIGTERM` signals, leading to abrupt exits.

**Code Before**:
```go
if runErr := ctx.Run(); runErr != nil {
    log.Printf("Service run error: %v\n", runErr)
    os.Exit(1)
}
```

**Improved Code**:
```go
stop := make(chan os.Signal, 1)
signal.Notify(stop, os.Interrupt, syscall.SIGTERM)

go func() {
    if runErr := ctx.Run(); runErr != nil {
        log.Printf("Service run error: %v\n", runErr)
        os.Exit(1)
    }
}()
<-stop
log.Println("Received shutdown signal, shutting down gracefully...")
```

---

## 4. Non-Critical Bugs

### 4.1 Hardcoded File Paths
- **Location**: [`/service/image.go`](https://github.com/alphabatem/nft-proxy/blob/main/service/image.go)
- **Issue**: Using hardcoded paths `./cache/solana/`, `./docs/failed_image.jpg` reduces flexibility. If you deploy to a different environment, you must edit the code instead of just changing configuration.

**Code Before**:
```go
cacheName := fmt.Sprintf("./cache/solana/%s.%s", media.Mint, media.ImageType)
```

**Improved Code**:
```go
cacheDir := os.Getenv("CACHE_DIR")
if cacheDir == "" {
    cacheDir = "./cache/solana"
}
cacheName := fmt.Sprintf("%s/%s.%s", cacheDir, media.Mint, media.ImageType)
```

- **Location**: `/service/http.go`

**Code Before**:
```go
svc.defaultImage, err = ioutil.ReadFile("./docs/failed_image.jpg")
if err != nil {
    log.Fatal(err)
}
```

**Improved Code**:
```go
defaultImagePath := os.Getenv("DEFAULT_IMAGE_PATH")
if defaultImagePath == "" {
   defaultImagePath = "./docs/failed_image.jpg"
}
svc.defaultImage, err = ioutil.ReadFile(defaultImagePath)
if err != nil {
   return fmt.Errorf("failed to load default image: %w", err)
}
```

**Benefits**: The code becomes more portable and easier to configure in different deployment environments without modifying the source code.

### 4.2 `.env` File Handling
- **Location**: [`/runtime/main.go`](https://github.com/alphabatem/nft-proxy/blob/main/runtime/main.go)
- **Issue**: Missing `.env` file triggers a `log.Fatal(...)` which may be too aggressive if environment variables are set elsewhere.

**Code Before**:
```go
if err := godotenv.Load(); err != nil {
    log.Fatal("Error loading .env file")
}
```

**Improved Code**:
```go
if err := godotenv.Load(); err != nil {
   log.Println("Warning: no .env file found; using system environment variables.")
}
```

---

### 4.3 Channel Closure in [collectionLoader]
- **Location**: [`/cli/load_collection_images.go`](https://github.com/alphabatem/nft-proxy/blob/main/cli/load_collection_images.go)
- **Issue**: `metaDataIn`, `fileIn`, and `mediaIn` channels are created but never closed. This can lead to worker goroutines blocking forever once they finish consuming.

**Code Before**:
```go
go func() {
   for _, meta := range fetchMetaData() {
      l.metaDataIn <- meta
   }
}()
```

**Improved Code**:
```go
go func() {
   defer close(l.metaDataIn)
   defer close(l.fileDataIn)
   defer close(l.mediaIn)

   for _, meta := range fetchMetaData() {
      l.metaDataIn <- meta
   }
   for _, file := range fetchFileData() {
      l.fileDataIn <- file
   }

   for _, media := range fetchMediaData() {
      l.mediaIn <- media
   }
}()
```

---

### 4.4 Inconsistent Error Logging
- **Issue**: Errors and logs are sometimes written with `log.Printf`, `log.Println` leading to inconsistent formatting.

**Code Before**:
```go
log.Printf("Error: %v", err)
log.Println("Processing image")
```

**Improved Code**:
```go
log.SetFlags(log.Ldate | log.Ltime | log.Lshortfile)
log.Printf("[ERROR] %v", err)
log.Printf("[INFO] %s", media.ImageUri)
```

---

## 5. Optimize Code

### 5.1 Testing
- **Location**: [`/service/solana_test.go`](https://github.com/alphabatem/nft-proxy/blob/main/service/solana_test.go)
- **Reason**: No clear unit or integration tests for critical services.

**Improved Code**:
```go
func TestSolanaService_TokenData(t *testing.T) {
    mockRPCClient := &MockRPCClient{}
    svc := &SolanaService{client: mockRPCClient}

    mockRPCClient.On("GetMultipleAccountsWithOpts", mock.Anything, mock.Anything, mock.Anything).
        Return(mockAccountsResponse(), nil)

    metadata, decimals, err := svc.TokenData(mockPublicKey())
    assert.NoError(t, err)
    assert.NotNil(t, metadata)
    assert.Equal(t, uint8(2), decimals)
}
```

---

### 5.2 Service Decoupling
- **Location**: [`/service/image.go`](https://github.com/alphabatem/nft-proxy/blob/main/service/image.go)
- **Reason**: `ImageService` and `SolanaService` are tightly coupled, making it harder to test and scale.

**Code Before**:
```go
type ImageService struct {
    solanaService SolanaService
    db            Database
}

func (svc *ImageService) fetchMissingImage(media *nft_proxy.Media) error {
    tokenData, _, err := svc.solanaService.FetchTokenData(solana.MustPublicKeyFromBase58(media.Mint))
    if err != nil {
        return err
    }
    return svc.db.SaveMedia(tokenData.Media())
}
```

**Improved Code**:
```go
type Database interface {
SaveMedia(media *nft_proxy.Media) error
FetchMedia(hash string) (*nft_proxy.Media, error)
}

type Blockchain interface {
FetchTokenData(key solana.PublicKey) (*token_metadata.Metadata, uint8, error)
}

type ImageService struct {
db        Database
blockchain Blockchain
}

func (svc *ImageService) fetchMissingImage(media *nft_proxy.Media) error {
tokenData, _, err := svc.blockchain.FetchTokenData(solana.MustPublicKeyFromBase58(media.Mint))
if err != nil {
return err
}
return svc.db.SaveMedia(tokenData.Media())
}
```

---

For the full details, you can refer to the [repository link](https://github.com/alphabatem/nft-proxy.git).
