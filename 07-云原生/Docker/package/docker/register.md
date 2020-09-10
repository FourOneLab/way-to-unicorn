---
title: package register
date: 2020-04-14T10:09:14.126627+08:00
draft: false
---

```bash
import "github.com/docker/docker/registry"
```

## Constants

```go
const (
    // AuthClientID用于令牌服务器使用的ClientID
    AuthClientID = "docker"

       // DefaultSearchLimit是返回的最大搜索结果数的默认值
    DefaultSearchLimit = 25
)
```

## Variables

```go
var (
    // DefaultNamespace是默认名称空间
    DefaultNamespace = "docker.io"

    // DefaultRegistryVersionHeader是带有注册表版本信息的默认HTTP标头的名称
    DefaultRegistryVersionHeader = "Docker-Distribution-Api-Version"

    // IndexHostname是索引主机名
    IndexHostname = "index.docker.io"

    // IndexServer用于用户身份验证和镜像搜索
    IndexServer = "https://" + IndexHostname + "/v1/"

    // IndexName是索引的名称
    IndexName = "docker.io"

    // DefaultV2Registry是默认v2注册表的URI
    DefaultV2Registry = &url.URL{
        Scheme: "https",
        Host:   "registry-1.docker.io",
    }

    // CertsDir是存储证书的目录
    CertsDir = "/etc/docker/certs.d"

    // 如果在远端已存在要推送的镜像，则返回ErrAlreadyExists错误
    ErrAlreadyExists = errors.New("Image already exists")

    // 如果存储库名称格式不正确，则返回ErrInvalidRepositoryName错误
    ErrInvalidRepositoryName = errors.New("Invalid repository name (ex: \"registry.domain.tld/myrepos\")")

    // 如果存储库在远程端不存在，则返回ErrRepoNotFound
    ErrRepoNotFound notFoundError = "Repository not found"
)
```

## func AuthTransport

```go
func AuthTransport(base http.RoundTripper, authConfig *types.AuthConfig, alwaysSetBasicAuth bool) http.RoundTripper
```

与v1注册表（私有或官方）通信时，AuthTransport处理auth层。

对于私有v1注册表，请将`alwaysSetBasicAuth`设置为true。

对于官方v1注册表，如果请求中没有Authorization标头，但是`X-Docker-Token`标头设置为true，则将使用Basic Auth设置Authorization标头。 使用提供的基本`http.RoundTripper`发送请求之后，如果响应中存在表示令牌的`X-Docker-Token`标头，则将其缓存并发送到所有后续请求的Authorization标头中。

如果服务器在客户端未请求的情况下发送令牌，则将其忽略。

此RoundTripper还具有CancelRequest方法，对于正确处理超时很重要。

## func ConvertToHostname

```go
func ConvertToHostname(url string) string
```

ConvertToHostname将前缀为`http|https`的注册表URL转换为主机名。

## func GetAuthConfigKey

```go
func GetAuthConfigKey(index *registrytypes.IndexInfo) string
```

GetAuthConfigKey特殊情况下使用正式索引的完整索引地址作为AuthConfig密钥，并将(host)name[:port]用于私有索引。

## func HTTPClient

```go
func HTTPClient(transport http.RoundTripper) *http.Client
```

HTTPClient返回一个HTTP客户端结构体，该结构体使用给定的传输方式，并包含重定向请求所需的标头。

## func Headers

```go
func Headers(userAgent string, metaHeaders http.Header) []transport.RequestModifier
```

Header返回带有User-Agent和metaHeaders的请求修饰符。

## func NewStaticCredentialStore

```go
func NewStaticCredentialStore(auth *types.AuthConfig) auth.CredentialStore
```

NewStaticCredentialStore返回一个凭证存储，该存储始终返回相同的凭证值。

## func NewTransport

```go
func NewTransport(tlsConfig *tls.Config) *http.Transport
```

NewTransport返回新的HTTP传输。如果tlsConfig为nil，则使用默认的TLS配置。

## func ParseSearchIndexInfo

```go
func ParseSearchIndexInfo(reposName string) (*registrytypes.IndexInfo, error)
```

ParseSearchIndexInfo将使用存储库名称来获取indexInfo。

## func PingV2Registry

```go
func PingV2Registry(endpoint *url.URL, transport http.RoundTripper) (challenge.Manager, bool, error)
```

PingV2Registry尝试对v2注册表执行ping操作，并在成功后返回质询管理器以获取受支持的身份验证类型以及响应是否确认了v2。 如果收到响应但无法解释，则将返回PingResponseError。

## func ReadCertsDirectory

```go
func ReadCertsDirectory(tlsConfig *tls.Config, directory string) error
```

ReadCertsDirectory读取TLS证书（包括根和证书对）的目录，并更新提供的TLS配置。

## func ResolveAuthConfig

```go
func ResolveAuthConfig(authConfigs map[string]types.AuthConfig, index *registrytypes.IndexInfo) types.AuthConfig
```

ResolveAuthConfig将身份验证配置与服务器地址或URL匹配。

## func ValidateIndexName

```go
func ValidateIndexName(val string) (string, error)
```

ValidateIndexName验证索引名称。

## func ValidateMirror

```go
func ValidateMirror(val string) (string, error)
```

ValidateMirror验证HTTP(s)注册表镜像。

## type APIEndpoint

```go
type APIEndpoint struct {
    Mirror                         bool
    URL                            *url.URL
    Version                        APIVersion
    AllowNondistributableArtifacts bool
    Official                       bool
    TrimHostname                   bool
    TLSConfig                      *tls.Config
}
```

APIEndpoint代表远程API端点。

### func (APIEndpoint) ToV1Endpoint

```go
func (e APIEndpoint) ToV1Endpoint(userAgent string, metaHeaders http.Header) *V1Endpoint
```

ToV1Endpoint返回基于APIEndpoint的V1 API端点。

## type APIVersion

```go
type APIVersion int
```

APIVersion是API版本（目前为1或2）的完整表示。

```go
const (
    APIVersion1 APIVersion = iota
    APIVersion2
)
```

API版本标识符。

### func (APIVersion) String

```go
func (av APIVersion) String() string
```

## type DefaultService

```go
type DefaultService struct {
    config *serviceConfig
    mu     sync.Mutex
}
```

DefaultService是一个注册表服务。它跟踪配置数据，例如镜像列表。

### func NewService

```go
func NewService(options ServiceOptions) (*DefaultService, error)
```

NewService返回一个DefaultService的新实例，该实例准备安装到引擎中。

### func (*DefaultService) Auth

```go
func (s *DefaultService) Auth(ctx context.Context, authConfig *types.AuthConfig, userAgent string) (status, token string, err error)
```

Auth使用提供的凭据联系公共注册表，如果身份验证成功，则返回OK。它可以用来验证客户凭证的有效性。

### func (*DefaultService) LoadAllowNondistributableArtifacts

```go
func (s *DefaultService) LoadAllowNondistributableArtifacts(registries []string) error
```

LoadAllowNondistributableArtifacts会加载`allow-nondistributable-artifacts`注册表到服务。

### func (*DefaultService) LoadInsecureRegistries

```go
func (s *DefaultService) LoadInsecureRegistries(registries []string) error
```

LoadInsecureRegistries加载服务的不安全注册表。

### func (*DefaultService) LoadMirrors

```go
func (s *DefaultService) LoadMirrors(mirrors []string) error
```

LoadMirrors为服务加载注册表镜像。

### func (*DefaultService) LookupPullEndpoints

```go
func (s *DefaultService) LookupPullEndpoints(hostname string) (endpoints []APIEndpoint, err error)
```

LookupPullEndpoints按照优先级创建一个端点列表来尝试拉取镜像。优先选择V2的端点，优先使用镜像地址，优先使用HTTPS协议。

### func (*DefaultService) LookupPushEndpoints

```go
func (s *DefaultService) LookupPushEndpoints(hostname string) (endpoints []APIEndpoint, err error)
```

LookupPushEndpoints按照优先级创建端点列表来尝试推送镜像。优先选择V2的端点，优先使用HTTPS协议，镜像地址不会被使用。

### func (*DefaultService) ResolveRepository

```go
func (s *DefaultService) ResolveRepository(name reference.Named) (*RepositoryInfo, error)
```

ResolveRepository将存储库名称拆分为其组件和所关联的注册表。

### func (*DefaultService) Search

```go
func (s *DefaultService) Search(ctx context.Context, term string, limit int, authConfig *types.AuthConfig, userAgent string, headers map[string][]string) (*registrytypes.SearchResults, error)
```

Search在公共注册表中查询与指定搜索词匹配的镜像，然后返回结果。

### func (*DefaultService) ServiceConfig

```go
func (s *DefaultService) ServiceConfig() *registrytypes.ServiceConfig
```

ServiceConfig返回公共注册表服务的配置。

## func (*DefaultService) TLSConfig

```go
func (s *DefaultService) TLSConfig(hostname string) (*tls.Config, error)
```

TLSConfig根据服务器默认值构造客户端TLS配置。

## type ImgData

```go
type ImgData struct {
    // ID是标识镜像的的opaque字符串
    ID              string `json:"id"`
    Checksum        string `json:"checksum,omitempty"`
    ChecksumPayload string `json:"-"`
    Tag             string `json:",omitempty"`
}
```

ImgData用于将镜像校验和与注册表进行传输（发送或获取）。

## type PingResponseError

```go
type PingResponseError struct {
    Err error
}
```

当收到来自ping的响应但无效时，使用PingResponseError。

### func (PingResponseError) Error

```go
func (err PingResponseError) Error() string
```

## type PingResult

```go
type PingResult struct {
    // 版本是注册表在HTTP标头中提供的注册表版本
    Version string `json:"version"`
    // 如果注册表在X-Docker-Registry-Standalone标头中指示它是独立注册表，则将Standalone设置为true
    Standalone bool `json:"standalone"`
}
```

PingResult包含ping注册表时返回的信息。它指示注册表的版本以及注册表是否声明是独立注册表。

## type RepositoryData

```go
type RepositoryData struct {
    // ImgList 是存储库中镜像列表
    ImgList map[string]*ImgData
    // Endpoints 是X-Docker-Endpoints中返回的端点的列表
    Endpoints []string
}
```

RepositoryData跟踪镜像列表和存储库的端点列表。

## type RepositoryInfo

```go
type RepositoryInfo struct {
    Name reference.Named
    // 索引指向注册表信息
    Index *registrytypes.IndexInfo
    // Official指示存储库是否被认为是官方的。 如果注册表是官方的，并且规范化名称中不包含'/'（例如“foo”），则将其视为正式的存储库。
    Official bool
    // Class表示存储库的类，例如“插件”或“镜像”。
    Class string
}
```

RepositoryInfo描述一个存储库。

### func ParseRepositoryInfo

```go
func ParseRepositoryInfo(reposName reference.Named) (*RepositoryInfo, error)
```

ParseRepositoryInfo将存储库名称分解为RepositoryInfo，但是缺少注册表信息。

## type Service

```go
type Service interface {
    Auth(ctx context.Context, authConfig *types.AuthConfig, userAgent string) (status, token string, err error)
    LookupPullEndpoints(hostname string) (endpoints []APIEndpoint, err error)
    LookupPushEndpoints(hostname string) (endpoints []APIEndpoint, err error)
    ResolveRepository(name reference.Named) (*RepositoryInfo, error)
    Search(ctx context.Context, term string, limit int, authConfig *types.AuthConfig, userAgent string, headers map[string][]string) (*registrytypes.SearchResults, error)
    ServiceConfig() *registrytypes.ServiceConfig
    TLSConfig(hostname string) (*tls.Config, error)
    LoadAllowNondistributableArtifacts([]string) error
    LoadMirrors([]string) error
    LoadInsecureRegistries([]string) error
}
```

Service是定义注册表服务应实现的接口。

## type ServiceOptions

```go
type ServiceOptions struct {
    AllowNondistributableArtifacts []string `json:"allow-nondistributable-artifacts,omitempty"`
    Mirrors                        []string `json:"registry-mirrors,omitempty"`
    InsecureRegistries             []string `json:"insecure-registries,omitempty"`
}
```

ServiceOptions保存命令行选项。

## type Session

```go
type Session struct {
    indexEndpoint *V1Endpoint
    client        *http.Client
    // TODO(tiborvass): remove authConfig
    authConfig *types.AuthConfig
    id         string
}
```

Session用于与V1注册表通信。

### func NewSession

```go
func NewSession(client *http.Client, authConfig *types.AuthConfig, endpoint *V1Endpoint) (*Session, error)
```

NewSession创建一个新的会话。

TODO（tiborvass）：一旦提供了注册表客户端v2，就删除authConfig参数。

### func (*Session) GetRemoteHistory

```go
func (r *Session) GetRemoteHistory(imgID, registry string) ([]string, error)
```

GetRemoteHistory从注册表中检索给定镜像的历史记录。它返回父级的JSON文件列表（包括请求的镜像）。

### func (*Session) GetRemoteImageJSON

```go
func (r *Session) GetRemoteImageJSON(imgID, registry string) ([]byte, int64, error)
```

GetRemoteImageJSON从注册表中检索镜像的JSON元数据。

### func (*Session) GetRemoteImageLayer

```go
func (r *Session) GetRemoteImageLayer(imgID, registry string, imgSize int64) (io.ReadCloser, error)
```

GetRemoteImageLayer从注册表中检索镜像层。

### func (*Session) GetRemoteTag

```go
func (r *Session) GetRemoteTag(registries []string, repositoryRef reference.Named, askedTag string) (string, error)
```

GetRemoteTag从给定的存储库中检索AskedTag参数中命名的标签。它查询registries参数中提供的每个注册表，并从第一个成功回答查询的注册表中返回数据。

### func (*Session) GetRemoteTags

```go
func (r *Session) GetRemoteTags(registries []string, repositoryRef reference.Named) (map[string]string, error)
```

GetRemoteTags从给定的存储库中检索所有标签。 它查询registries参数中提供的每个注册表，并从第一个成功回答查询的注册表中返回数据。 它返回一个以标签名作为键，并以镜像ID作为值的映射。

### func (*Session) GetRepositoryData

```go
func (r *Session) GetRepositoryData(name reference.Named) (*RepositoryData, error)
```

GetRepositoryData返回该存储库的镜像和端点的列表。

### func (*Session) ID

```go
func (r *Session) ID() string
```

D返回此注册表会话的ID。

### func (*Session) LookupRemoteImage

```go
func (r *Session) LookupRemoteImage(imgID, registry string) error
```

LookupRemoteImage检查注册表中是否存在镜像。

### func (*Session) PushImageChecksumRegistry

```go
func (r *Session) PushImageChecksumRegistry(imgData *ImgData, registry string) error
```

PushImageChecksumRegistry上传镜像的校验和。

### func (*Session) PushImageJSONIndex

```go
func (r *Session) PushImageJSONIndex(remote reference.Named, imgList []*ImgData, validate bool, regs []string) (*RepositoryData, error)
```

PushImageJSONIndex将镜像列表上传到存储库。

### func (*Session) PushImageJSONRegistry

```go
func (r *Session) PushImageJSONRegistry(imgData *ImgData, jsonRaw []byte, registry string) error
```

PushImageJSONRegistry将本地镜像的JSON元数据推送到注册表。

### func (*Session) PushImageLayerRegistry

```go
func (r *Session) PushImageLayerRegistry(imgID string, layer io.Reader, registry string, jsonRaw []byte) (chec
ksum string, checksumPayload string, err error)
```

PushImageLayerRegistry将镜像层的校验和发送到注册表。

### func (*Session) PushRegistryTag

```go
func (r *Session) PushRegistryTag(remote reference.Named, revision, tag, registry string) error
```

PushRegistryTag将标签推送到注册表中。远端获取到的格式为`<user>/<repo>`。

### func (*Session) SearchRepositories

```go
func (r *Session) SearchRepositories(term string, limit int) (*registrytypes.SearchResults, error)
```

SearchRepositories对远程存储库执行搜索。

## type V1Endpoint

```go
type V1Endpoint struct {
    client   *http.Client
    URL      *url.URL
    IsSecure bool
}
```

V1Endpoint存储有关V1注册表端点的基本信息。

### func NewV1Endpoint

```go
func NewV1Endpoint(index *registrytypes.IndexInfo, userAgent string, metaHeaders http.Header) (*V1Endpoint, error)
```

NewV1Endpoint解析给定地址以返回注册表端点。

### func (*V1Endpoint) Path

```go
func (e *V1Endpoint) Path(path string) string
```

Path返回此端点的URL的格式化字符串，并附加给定的路径。

### func (*V1Endpoint) Ping

```go
func (e *V1Endpoint) Ping() (PingResult, error)
```

Ping返回一个PingResult，该结果指示注册表是否独立。

### func (*V1Endpoint) String

```go
func (e *V1Endpoint) String() string
```

获取此注册表端点的root的格式化URL。
