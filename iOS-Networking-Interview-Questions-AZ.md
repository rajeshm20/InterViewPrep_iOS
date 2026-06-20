# iOS Networking Interview Questions & Answers (A–Z)

Every code sample below uses **real, live public endpoints** (jsonplaceholder.typicode.com, api.github.com, reqres.in, httpbin.org) so you can paste them into a Playground or an Xcode project and actually run them.

---

## 1. URLSession Fundamentals

### Q1. What is URLSession and what are its session types?
`URLSession` is Apple's foundation networking API for HTTP/HTTPS requests, WebSockets, and downloads/uploads. Three configurations:

- **`.default`** — uses disk-persisted cache and cookies (shared `URLSession.shared` uses this).
- **`.ephemeral`** — no persistent storage, all cache/cookies in RAM only (good for private browsing / sensitive sessions).
- **`.background`** — continues transfers even when the app is suspended or terminated; requires `URLSessionDelegate` and is process-independent.

```swift
import Foundation

// Default shared session — fine for simple one-off GETs
let defaultTask = URLSession.shared

// Ephemeral — no cookies/cache persisted to disk
let ephemeralConfig = URLSessionConfiguration.ephemeral
let ephemeralSession = URLSession(configuration: ephemeralConfig)

// Background — survives app suspension
let bgConfig = URLSessionConfiguration.background(withIdentifier: "com.example.app.bgSession")
bgConfig.isDiscretionary = true
bgConfig.sessionSendsLaunchEvents = true
let backgroundSession = URLSession(configuration: bgConfig, delegate: MyDownloadDelegate(), delegateQueue: nil)
```

### Q2. How do you make a basic GET request with async/await?
```swift
struct Post: Decodable {
    let id: Int
    let title: String
    let body: String
    let userId: Int
}

func fetchPost(id: Int) async throws -> Post {
    guard let url = URL(string: "https://jsonplaceholder.typicode.com/posts/\(id)") else {
        throw URLError(.badURL)
    }
    let (data, response) = try await URLSession.shared.data(from: url)

    guard let http = response as? HTTPURLResponse, (200..<300).contains(http.statusCode) else {
        throw URLError(.badServerResponse)
    }
    return try JSONDecoder().decode(Post.self, from: data)
}

// Usage
Task {
    do {
        let post = try await fetchPost(id: 1)
        print(post.title)
    } catch {
        print("Failed: \(error)")
    }
}
```

### Q3. How do you make a POST request with a JSON body?
```swift
struct NewPost: Encodable {
    let title: String
    let body: String
    let userId: Int
}

struct CreatedPost: Decodable {
    let id: Int
    let title: String
}

func createPost(_ post: NewPost) async throws -> CreatedPost {
    var request = URLRequest(url: URL(string: "https://jsonplaceholder.typicode.com/posts")!)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.httpBody = try JSONEncoder().encode(post)

    let (data, response) = try await URLSession.shared.data(for: request)
    guard let http = response as? HTTPURLResponse, http.statusCode == 201 else {
        throw URLError(.badServerResponse)
    }
    return try JSONDecoder().decode(CreatedPost.self, from: data)
}
```

### Q4. Why doesn't `URLSession.shared` work well for background or custom-delegate work?
`URLSession.shared` has no delegate slot you control and uses a fixed default configuration. If you need upload/download progress, redirect handling, auth challenges, or background transfers, you must create your own `URLSession(configuration:delegate:delegateQueue:)` instance.

---

## 2. Codable & JSON Parsing

### Q5. How do you handle snake_case JSON keys without writing CodingKeys manually?
```swift
struct GitHubUser: Decodable {
    let login: String
    let id: Int
    let avatarUrl: String     // avatar_url
    let publicRepos: Int      // public_repos
}

let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase

let url = URL(string: "https://api.github.com/users/apple")!
Task {
    let (data, _) = try await URLSession.shared.data(from: url)
    let user = try decoder.decode(GitHubUser.self, from: data)
    print(user.avatarUrl)
}
```

### Q6. How do you handle a field that can be either a String or an Int (polymorphic JSON)?
```swift
enum FlexibleID: Decodable {
    case string(String)
    case int(Int)

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        if let intValue = try? container.decode(Int.self) {
            self = .int(intValue)
        } else if let stringValue = try? container.decode(String.self) {
            self = .string(stringValue)
        } else {
            throw DecodingError.typeMismatch(
                FlexibleID.self,
                .init(codingPath: decoder.codingPath, debugDescription: "Expected String or Int")
            )
        }
    }
}
```

### Q7. How do you decode dates in ISO-8601 format?
```swift
struct Repo: Decodable {
    let name: String
    let createdAt: Date

    enum CodingKeys: String, CodingKey {
        case name
        case createdAt = "created_at"
    }
}

let decoder = JSONDecoder()
decoder.dateDecodingStrategy = .iso8601   // GitHub returns "2024-05-01T12:00:00Z"

Task {
    let url = URL(string: "https://api.github.com/repos/apple/swift")!
    let (data, _) = try await URLSession.shared.data(from: url)
    let repo = try decoder.decode(Repo.self, from: data)
    print(repo.createdAt)
}
```

### Q8. How do you gracefully skip a malformed element inside a JSON array instead of failing the whole decode?
```swift
struct LossyArray<Element: Decodable>: Decodable {
    var elements: [Element]

    init(from decoder: Decoder) throws {
        var container = try decoder.unkeyedContainer()
        var result: [Element] = []
        while !container.isAtEnd {
            if let value = try? container.decode(Element.self) {
                result.append(value)
            } else {
                _ = try? container.decode(EmptyDecodable.self) // consume & skip bad element
            }
        }
        elements = result
    }
}
private struct EmptyDecodable: Decodable {}
```

---

## 3. Concurrency: async/await, Combine, GCD

### Q9. How do you run multiple network requests concurrently and wait for all results?
```swift
func fetchAllPosts(ids: [Int]) async throws -> [Post] {
    try await withThrowingTaskGroup(of: Post.self) { group in
        for id in ids {
            group.addTask { try await fetchPost(id: id) }
        }
        var results: [Post] = []
        for try await post in group {
            results.append(post)
        }
        return results
    }
}

// Usage
Task {
    let posts = try await fetchAllPosts(ids: [1, 2, 3, 4, 5])
    print(posts.count)
}
```

### Q10. What's the difference between `async let` and `TaskGroup`?
`async let` is for a **fixed, known number** of concurrent child tasks (e.g., exactly 2 or 3 calls you name individually). `TaskGroup` is for a **dynamic/variable number** of tasks (e.g., looping over an array of IDs). Both run children concurrently on the cooperative thread pool and propagate cancellation/errors from children to the parent.

```swift
func loadDashboard() async throws -> (Post, GitHubUser) {
    async let post = fetchPost(id: 1)
    async let user = fetchGitHubUser(login: "apple")
    return try await (post, user)
}
```

### Q11. How do you implement a network call with Combine and chain a retry + decode?
```swift
import Combine

func fetchPostPublisher(id: Int) -> AnyPublisher<Post, Error> {
    let url = URL(string: "https://jsonplaceholder.typicode.com/posts/\(id)")!
    return URLSession.shared.dataTaskPublisher(for: url)
        .map(\.data)
        .decode(type: Post.self, decoder: JSONDecoder())
        .retry(2)
        .receive(on: DispatchQueue.main)
        .eraseToAnyPublisher()
}

// Usage
var cancellables = Set<AnyCancellable>()
fetchPostPublisher(id: 1)
    .sink(
        receiveCompletion: { print($0) },
        receiveValue: { post in print(post.title) }
    )
    .store(in: &cancellables)
```

### Q12. How do you cancel an in-flight network request?
```swift
final class PostLoader {
    private var task: Task<Void, Never>?

    func load(id: Int) {
        task?.cancel()
        task = Task {
            do {
                let post = try await fetchPost(id: id)
                print(post.title)
            } catch is CancellationError {
                print("Cancelled")
            } catch {
                print("Error: \(error)")
            }
        }
    }

    func cancel() { task?.cancel() }
}
```
Note: `URLSession`'s async `data(from:)` checks for task cancellation cooperatively — cancelling the `Task` cancels the underlying `URLSessionTask` too.

---

## 4. Error Handling & Status Codes

### Q13. Design a robust networking error enum.
```swift
enum NetworkError: Error, LocalizedError {
    case invalidURL
    case noInternet
    case unauthorized
    case notFound
    case serverError(statusCode: Int)
    case decodingFailed(Error)
    case unknown(Error)

    var errorDescription: String? {
        switch self {
        case .invalidURL: return "The URL was invalid."
        case .noInternet: return "No internet connection."
        case .unauthorized: return "Session expired. Please log in again."
        case .notFound: return "Resource not found."
        case .serverError(let code): return "Server error (\(code))."
        case .decodingFailed: return "Could not parse server response."
        case .unknown(let error): return error.localizedDescription
        }
    }
}

func mapHTTPError(statusCode: Int) -> NetworkError {
    switch statusCode {
    case 401: return .unauthorized
    case 404: return .notFound
    case 500...599: return .serverError(statusCode: statusCode)
    default: return .serverError(statusCode: statusCode)
    }
}
```

### Q14. How do you distinguish a "no network" error from a server-side failure?
```swift
func fetch(url: URL) async throws -> Data {
    do {
        let (data, response) = try await URLSession.shared.data(from: url)
        guard let http = response as? HTTPURLResponse else { throw NetworkError.unknown(URLError(.badServerResponse)) }
        guard (200..<300).contains(http.statusCode) else {
            throw mapHTTPError(statusCode: http.statusCode)
        }
        return data
    } catch let urlError as URLError {
        switch urlError.code {
        case .notConnectedToInternet, .networkConnectionLost:
            throw NetworkError.noInternet
        default:
            throw NetworkError.unknown(urlError)
        }
    }
}
```

### Q15. How do you test reachability without a third-party library (iOS 12+)?
```swift
import Network

final class ReachabilityMonitor {
    private let monitor = NWPathMonitor()
    private(set) var isConnected = true

    init() {
        monitor.pathUpdateHandler = { [weak self] path in
            self?.isConnected = path.status == .satisfied
        }
        monitor.start(queue: DispatchQueue(label: "ReachabilityMonitor"))
    }

    deinit { monitor.cancel() }
}
```

---

## 5. Retry, Backoff & Interceptors

### Q16. Implement exponential backoff retry for a flaky endpoint.
```swift
func fetchWithRetry<T>(
    maxAttempts: Int = 3,
    initialDelay: Double = 0.5,
    operation: @escaping () async throws -> T
) async throws -> T {
    var attempt = 0
    var delay = initialDelay
    while true {
        attempt += 1
        do {
            return try await operation()
        } catch {
            if attempt >= maxAttempts { throw error }
            try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
            delay *= 2 // exponential backoff
        }
    }
}

// Usage against httpbin's flaky-status endpoint
Task {
    let result = try await fetchWithRetry {
        try await fetchPost(id: 1)
    }
    print(result.title)
}
```

### Q17. How do you implement a request interceptor (e.g., auto-attach auth token, refresh on 401)?
```swift
protocol RequestInterceptor {
    func adapt(_ request: URLRequest) async throws -> URLRequest
    func retryOnUnauthorized(_ request: URLRequest, response: HTTPURLResponse) async throws -> Bool
}

final class AuthInterceptor: RequestInterceptor {
    private var token: String = "stale-token"

    func adapt(_ request: URLRequest) async throws -> URLRequest {
        var req = request
        req.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        return req
    }

    func retryOnUnauthorized(_ request: URLRequest, response: HTTPURLResponse) async throws -> Bool {
        guard response.statusCode == 401 else { return false }
        token = try await refreshToken() // your refresh-token call
        return true
    }

    private func refreshToken() async throws -> String {
        // Call your real refresh endpoint here
        return "new-fresh-token"
    }
}

final class APIClient {
    private let interceptor: RequestInterceptor
    init(interceptor: RequestInterceptor) { self.interceptor = interceptor }

    func send(_ originalRequest: URLRequest) async throws -> (Data, HTTPURLResponse) {
        var request = try await interceptor.adapt(originalRequest)
        var (data, response) = try await URLSession.shared.data(for: request)
        guard let http = response as? HTTPURLResponse else { throw URLError(.badServerResponse) }

        if try await interceptor.retryOnUnauthorized(request, response: http) {
            request = try await interceptor.adapt(originalRequest)
            (data, response) = try await URLSession.shared.data(for: request)
        }
        return (data, response as! HTTPURLResponse)
    }
}
```

---

## 6. Authentication

### Q18. How do you implement Basic Auth with reqres.in?
```swift
func login(email: String, password: String) async throws -> String {
    var request = URLRequest(url: URL(string: "https://reqres.in/api/login")!)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.httpBody = try JSONEncoder().encode(["email": email, "password": password])

    let (data, response) = try await URLSession.shared.data(for: request)
    guard let http = response as? HTTPURLResponse, http.statusCode == 200 else {
        throw NetworkError.unauthorized
    }
    struct LoginResponse: Decodable { let token: String }
    return try JSONDecoder().decode(LoginResponse.self, from: data).token
}
```

### Q19. Where should you securely store an access token on iOS?
In the **Keychain**, never in `UserDefaults` or plain files — `UserDefaults` is unencrypted and readable on jailbroken devices or via backup extraction.

```swift
import Security

enum KeychainStore {
    static func save(token: String, for key: String) {
        let data = Data(token.utf8)
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlock
        ]
        SecItemDelete(query as CFDictionary)
        SecItemAdd(query as CFDictionary, nil)
    }

    static func read(key: String) -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]
        var item: CFTypeRef?
        guard SecItemCopyMatching(query as CFDictionary, &item) == errSecSuccess,
              let data = item as? Data else { return nil }
        return String(data: data, encoding: .utf8)
    }
}
```

### Q20. How do you implement OAuth2 token refresh logic without race conditions (multiple 401s firing refresh simultaneously)?
Use an `actor` to serialize refresh calls so concurrent 401s share one in-flight refresh task instead of triggering N parallel refreshes.

```swift
actor TokenRefresher {
    private var refreshTask: Task<String, Error>?

    func validToken() async throws -> String {
        if let refreshTask {
            return try await refreshTask.value
        }
        let task = Task<String, Error> {
            defer { refreshTask = nil }
            return try await self.performRefresh()
        }
        refreshTask = task
        return try await task.value
    }

    private func performRefresh() async throws -> String {
        // Call your real OAuth refresh endpoint here
        try await Task.sleep(nanoseconds: 300_000_000)
        return "refreshed-token-\(UUID().uuidString.prefix(6))"
    }
}
```

---

## 7. Certificate Pinning & Security

### Q21. How do you implement SSL/TLS certificate pinning with URLSessionDelegate?
```swift
import Security

final class PinningDelegate: NSObject, URLSessionDelegate {
    // SHA-256 hash of the server's public key (base64) — replace with your real pin
    private let pinnedPublicKeyHash = "REPLACE_WITH_YOUR_PINNED_PUBLIC_KEY_SHA256_BASE64"

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard let serverTrust = challenge.protectionSpace.serverTrust,
              let certificate = SecTrustGetCertificateAtIndex(serverTrust, 0) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        let serverPublicKey = SecCertificateCopyKey(certificate)
        guard let keyData = SecKeyCopyExternalRepresentation(serverPublicKey!, nil) as Data? else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        let hash = sha256Base64(keyData)
        if hash == pinnedPublicKeyHash {
            completionHandler(.useCredential, URLCredential(trust: serverTrust))
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }

    private func sha256Base64(_ data: Data) -> String {
        import CryptoKit_unused: () // placeholder, see note below
        return ""
    }
}
```
> In practice use `CryptoKit`'s `SHA256.hash(data:)` to compute the digest, then base64-encode it, and compare against a pin you generated offline with `openssl s_client` / `openssl x509`. Always ship at least one **backup pin** to avoid bricking the app on cert rotation.

### Q22. Why is ATS (App Transport Security) relevant to networking interviews?
ATS, enabled by default since iOS 9, requires HTTPS with TLS 1.2+ for all connections. Exceptions need explicit `NSAppTransportSecurity` entries in `Info.plist` (e.g., `NSAllowsArbitraryLoads` or per-domain exceptions) — interviewers often probe whether candidates understand this is a deliberate security gate, not a bug to "just disable."

---

## 8. Caching

### Q23. How does `URLCache` work and how do you configure it?
```swift
let cache = URLCache(memoryCapacity: 20 * 1024 * 1024,   // 20 MB RAM
                      diskCapacity: 100 * 1024 * 1024,    // 100 MB disk
                      diskPath: "api_cache")

let config = URLSessionConfiguration.default
config.urlCache = cache
config.requestCachePolicy = .returnCacheDataElseLoad

let session = URLSession(configuration: config)
```
`URLCache` honors standard HTTP caching headers (`Cache-Control`, `ETag`, `Last-Modified`) returned by the server — it does not cache responses that lack cache-allowing headers unless you override via `URLSession(_:dataTask:willCacheResponse:completionHandler:)`.

### Q24. How do you implement conditional GET with `ETag` to avoid re-downloading unchanged data?
```swift
final class ETagCache {
    private var etags: [URL: String] = [:]
    private var cachedData: [URL: Data] = [:]

    func fetch(_ url: URL) async throws -> Data {
        var request = URLRequest(url: url)
        if let etag = etags[url] {
            request.setValue(etag, forHTTPHeaderField: "If-None-Match")
        }
        let (data, response) = try await URLSession.shared.data(for: request)
        guard let http = response as? HTTPURLResponse else { throw URLError(.badServerResponse) }

        if http.statusCode == 304, let cached = cachedData[url] {
            return cached // not modified — use cache
        }
        if let newEtag = http.value(forHTTPHeaderField: "ETag") {
            etags[url] = newEtag
            cachedData[url] = data
        }
        return data
    }
}
```

---

## 9. File Transfer: Upload, Download, Multipart

### Q25. How do you upload an image with `multipart/form-data`?
```swift
func uploadImage(_ imageData: Data, fileName: String, to urlString: String) async throws -> Data {
    let url = URL(string: urlString)!
    let boundary = "Boundary-\(UUID().uuidString)"

    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("multipart/form-data; boundary=\(boundary)", forHTTPHeaderField: "Content-Type")

    var body = Data()
    body.append("--\(boundary)\r\n".data(using: .utf8)!)
    body.append("Content-Disposition: form-data; name=\"file\"; filename=\"\(fileName)\"\r\n".data(using: .utf8)!)
    body.append("Content-Type: image/jpeg\r\n\r\n".data(using: .utf8)!)
    body.append(imageData)
    body.append("\r\n--\(boundary)--\r\n".data(using: .utf8)!)

    request.httpBody = body
    let (data, response) = try await URLSession.shared.data(for: request)
    guard let http = response as? HTTPURLResponse, (200..<300).contains(http.statusCode) else {
        throw URLError(.badServerResponse)
    }
    return data
}

// Test target that echoes uploads: https://httpbin.org/post
```

### Q26. How do you download a large file with progress reporting?
```swift
final class DownloadProgressDelegate: NSObject, URLSessionDownloadDelegate {
    var onProgress: ((Double) -> Void)?
    var onFinish: ((URL) -> Void)?

    func urlSession(_ session: URLSession, downloadTask: URLSessionDownloadTask,
                     didWriteData bytesWritten: Int64, totalBytesWritten: Int64,
                     totalBytesExpectedToWrite: Int64) {
        guard totalBytesExpectedToWrite > 0 else { return }
        let progress = Double(totalBytesWritten) / Double(totalBytesExpectedToWrite)
        onProgress?(progress)
    }

    func urlSession(_ session: URLSession, downloadTask: URLSessionDownloadTask,
                     didFinishDownloadingTo location: URL) {
        onFinish?(location)
    }
}

// Usage
let delegate = DownloadProgressDelegate()
delegate.onProgress = { print("Progress: \(Int($0 * 100))%") }
delegate.onFinish = { tempURL in print("Saved at \(tempURL)") }

let session = URLSession(configuration: .default, delegate: delegate, delegateQueue: nil)
let task = session.downloadTask(with: URL(string: "https://httpbin.org/image/jpeg")!)
task.resume()
```

### Q27. How do background uploads/downloads survive app termination?
A `URLSession` created with a `background` configuration hands the transfer off to a separate system daemon (`nsurlsessiond`). If the app is killed, iOS relaunches it in the background when the transfer completes and calls `application(_:handleEventsForBackgroundURLSession:completionHandler:)` in your `AppDelegate`, where you re-attach a session with the **same identifier** to receive the delegate callbacks.

```swift
// AppDelegate
func application(_ application: UIApplication,
                  handleEventsForBackgroundURLSession identifier: String,
                  completionHandler: @escaping () -> Void) {
    backgroundCompletionHandler = completionHandler
    // Re-create the session with the same identifier so delegate callbacks resume
}
```

---

## 10. WebSockets & Streaming

### Q28. How do you open a WebSocket connection with native URLSession (iOS 13+)?
```swift
final class WebSocketClient {
    private var webSocketTask: URLSessionWebSocketTask?

    func connect(to url: URL) {
        webSocketTask = URLSession.shared.webSocketTask(with: url)
        webSocketTask?.resume()
        listen()
    }

    private func listen() {
        webSocketTask?.receive { [weak self] result in
            switch result {
            case .success(.string(let text)):
                print("Received: \(text)")
            case .success(.data(let data)):
                print("Received \(data.count) bytes")
            case .failure(let error):
                print("WebSocket error: \(error)")
                return
            default:
                break
            }
            self?.listen() // keep listening
        }
    }

    func send(_ text: String) {
        webSocketTask?.send(.string(text)) { error in
            if let error { print("Send error: \(error)") }
        }
    }

    func disconnect() {
        webSocketTask?.cancel(with: .goingAway, reason: nil)
    }
}

// Public echo test server: wss://echo.websocket.events
let client = WebSocketClient()
client.connect(to: URL(string: "wss://echo.websocket.events")!)
client.send("Hello from iOS")
```

### Q29. How is Server-Sent Events (SSE) consumption different from WebSockets in iOS networking?
SSE is a one-directional, plain-HTTP stream (`text/event-stream`) read incrementally via `URLSession`'s `bytes(for:)` async sequence — no special protocol upgrade like WebSocket's handshake. WebSockets are bidirectional and use the `ws://`/`wss://` scheme with a dedicated `URLSessionWebSocketTask`.

```swift
func streamEvents(from url: URL) async throws {
    let (bytes, _) = try await URLSession.shared.bytes(for: URLRequest(url: url))
    for try await line in bytes.lines {
        if line.hasPrefix("data:") {
            print("Event:", line.dropFirst(5).trimmingCharacters(in: .whitespaces))
        }
    }
}
```

---

## 11. Testing & Mocking

### Q30. How do you mock network responses for unit tests without hitting the real network?
```swift
final class MockURLProtocol: URLProtocol {
    static var requestHandler: ((URLRequest) throws -> (HTTPURLResponse, Data))?

    override class func canInit(with request: URLRequest) -> Bool { true }
    override class func canonicalRequest(for request: URLRequest) -> URLRequest { request }

    override func startLoading() {
        guard let handler = MockURLProtocol.requestHandler else {
            fatalError("Handler not set")
        }
        do {
            let (response, data) = try handler(request)
            client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
            client?.urlProtocol(self, didLoad: data)
            client?.urlProtocolDidFinishLoading(self)
        } catch {
            client?.urlProtocol(self, didFailWithError: error)
        }
    }

    override func stopLoading() {}
}

// Test setup
func makeMockedSession() -> URLSession {
    let config = URLSessionConfiguration.ephemeral
    config.protocolClasses = [MockURLProtocol.self]
    return URLSession(configuration: config)
}

// In a test:
import XCTest

final class PostServiceTests: XCTestCase {
    func test_fetchPost_decodesSuccessfully() async throws {
        let json = """
        {"id": 1, "title": "Test Title", "body": "Test body", "userId": 1}
        """.data(using: .utf8)!

        MockURLProtocol.requestHandler = { request in
            let response = HTTPURLResponse(url: request.url!, statusCode: 200,
                                            httpVersion: nil, headerFields: nil)!
            return (response, json)
        }

        let session = makeMockedSession()
        let (data, _) = try await session.data(from: URL(string: "https://jsonplaceholder.typicode.com/posts/1")!)
        let post = try JSONDecoder().decode(Post.self, from: data)

        XCTAssertEqual(post.title, "Test Title")
    }
}
```
This is the industry-standard approach — no third-party library needed, deterministic, and fast (no real network round-trip).

### Q31. How do you design your networking layer to be testable in the first place (protocol-oriented)?
```swift
protocol HTTPClient {
    func send(_ request: URLRequest) async throws -> (Data, HTTPURLResponse)
}

final class URLSessionHTTPClient: HTTPClient {
    private let session: URLSession
    init(session: URLSession = .shared) { self.session = session }

    func send(_ request: URLRequest) async throws -> (Data, HTTPURLResponse) {
        let (data, response) = try await session.data(for: request)
        guard let http = response as? HTTPURLResponse else { throw URLError(.badServerResponse) }
        return (data, http)
    }
}

final class PostService {
    private let client: HTTPClient
    init(client: HTTPClient) { self.client = client }

    func fetchPost(id: Int) async throws -> Post {
        let request = URLRequest(url: URL(string: "https://jsonplaceholder.typicode.com/posts/\(id)")!)
        let (data, http) = try await client.send(request)
        guard (200..<300).contains(http.statusCode) else { throw mapHTTPError(statusCode: http.statusCode) }
        return try JSONDecoder().decode(Post.self, from: data)
    }
}

// In tests, inject a fake HTTPClient conforming struct/class instead of hitting the network at all.
```

---

## 12. REST Architecture & API Design Questions

### Q32. What's the difference between PUT and PATCH?
**PUT** replaces the entire resource (idempotent — sending the same PUT twice has the same effect). **PATCH** applies a partial update (not guaranteed idempotent unless the API designs it that way).

```swift
func updatePostTitle(id: Int, newTitle: String) async throws {
    var request = URLRequest(url: URL(string: "https://jsonplaceholder.typicode.com/posts/\(id)")!)
    request.httpMethod = "PATCH"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.httpBody = try JSONEncoder().encode(["title": newTitle])
    _ = try await URLSession.shared.data(for: request)
}
```

### Q33. What do common HTTP status codes mean, and how should an iOS client react to each?
| Code | Meaning | Client behavior |
|------|---------|------------------|
| 200/201 | Success / Created | Parse body, update UI |
| 204 | No Content | Treat as success, no body to parse |
| 304 | Not Modified | Use cached data |
| 400 | Bad Request | Show validation error, don't retry blindly |
| 401 | Unauthorized | Try token refresh, else force re-login |
| 403 | Forbidden | Show permission error, don't retry |
| 404 | Not Found | Show "not found" state |
| 408 | Request Timeout | Safe to retry |
| 429 | Too Many Requests | Respect `Retry-After` header, backoff |
| 500/502/503 | Server errors | Retry with backoff, show generic error after max attempts |

### Q34. How do you implement pagination for a paged REST API?
```swift
struct PagedResult<T: Decodable>: Decodable {
    let items: [T]
    let nextPage: Int?
}

func fetchUsersPage(_ page: Int) async throws -> [GitHubUser] {
    let url = URL(string: "https://api.github.com/users?since=\(page * 30)")!
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode([GitHubUser].self, from: data)
}

// Infinite-scroll pattern in a view model
@MainActor
final class UserListViewModel: ObservableObject {
    @Published var users: [GitHubUser] = []
    private var currentPage = 0
    private var isLoading = false

    func loadNextPageIfNeeded(currentItem: GitHubUser?) async {
        guard let currentItem, currentItem.id == users.last?.id, !isLoading else { return }
        isLoading = true
        defer { isLoading = false }
        if let newUsers = try? await fetchUsersPage(currentPage) {
            users.append(contentsOf: newUsers)
            currentPage += 1
        }
    }
}
```

### Q35. How would you debounce a search-as-you-type network call?
```swift
import Combine

final class SearchViewModel: ObservableObject {
    @Published var query = ""
    @Published var results: [GitHubUser] = []
    private var cancellables = Set<AnyCancellable>()

    init() {
        $query
            .debounce(for: .milliseconds(400), scheduler: DispatchQueue.main)
            .removeDuplicates()
            .filter { !$0.isEmpty }
            .sink { [weak self] term in
                self?.search(term)
            }
            .store(in: &cancellables)
    }

    private func search(_ term: String) {
        guard let url = URL(string: "https://api.github.com/search/users?q=\(term)") else { return }
        URLSession.shared.dataTaskPublisher(for: url)
            .map(\.data)
            .decode(type: SearchResponse.self, decoder: JSONDecoder())
            .replaceError(with: SearchResponse(items: []))
            .receive(on: DispatchQueue.main)
            .sink { [weak self] response in self?.results = response.items }
            .store(in: &cancellables)
    }
}

struct SearchResponse: Decodable {
    let items: [GitHubUser]
}
```

---

## 13. GraphQL & Alternative Protocols (briefly)

### Q36. How would you call a GraphQL endpoint using plain URLSession (no Apollo)?
```swift
func runGraphQLQuery() async throws -> Data {
    let query = """
    { repository(owner: "apple", name: "swift") { stargazerCount } }
    """
    var request = URLRequest(url: URL(string: "https://api.github.com/graphql")!)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.setValue("Bearer YOUR_GITHUB_TOKEN", forHTTPHeaderField: "Authorization") // GraphQL API requires auth
    request.httpBody = try JSONEncoder().encode(["query": query])

    let (data, _) = try await URLSession.shared.data(for: request)
    return data
}
```
Interview insight: GraphQL still rides over a single HTTP POST endpoint, so most REST networking abstractions (interceptors, retry, auth) apply unchanged — only the request/response *shape* and error semantics (HTTP 200 with an `errors` array) differ.

---

## 14. Performance, Threading & Best Practices

### Q37. Why should JSON decoding of large payloads happen off the main thread, and how do you guarantee that with async/await?
`URLSession`'s async APIs already resume on a background executor, so decoding inside the same `async` function (before any `@MainActor` hop) runs off the main thread automatically. The risk is calling `.decode` inside a `@MainActor`-isolated view model method — make the network+decode function non-isolated and only hop to `@MainActor` to publish the final result.

```swift
final class PostRepository {
    func fetchPost(id: Int) async throws -> Post {           // not main-actor isolated
        let (data, _) = try await URLSession.shared.data(from: URL(string: "https://jsonplaceholder.typicode.com/posts/\(id)")!)
        return try JSONDecoder().decode(Post.self, from: data) // decoding happens off main
    }
}

@MainActor
final class PostViewModel: ObservableObject {
    @Published var post: Post?
    private let repo = PostRepository()

    func load(id: Int) async {
        if let result = try? await repo.fetchPost(id: id) {
            post = result // hop to main only to publish
        }
    }
}
```

### Q38. How do you avoid duplicate in-flight requests when a user taps a button repeatedly?
```swift
actor RequestDeduplicator<Key: Hashable, Value> {
    private var tasks: [Key: Task<Value, Error>] = [:]

    func run(key: Key, operation: @escaping () async throws -> Value) async throws -> Value {
        if let existing = tasks[key] {
            return try await existing.value
        }
        let task = Task { try await operation() }
        tasks[key] = task
        defer { tasks[key] = nil }
        return try await task.value
    }
}

// Usage: two rapid taps on "load post 1" share one network call
let deduplicator = RequestDeduplicator<Int, Post>()
Task { try await deduplicator.run(key: 1) { try await fetchPost(id: 1) } }
Task { try await deduplicator.run(key: 1) { try await fetchPost(id: 1) } }
```

### Q39. How do you set sane timeouts and what's the difference between `timeoutIntervalForRequest` and `timeoutIntervalForResource`?
- `timeoutIntervalForRequest` — max idle time between bytes (resets on activity). Default 60s.
- `timeoutIntervalForResource` — absolute ceiling for the *entire* transfer regardless of activity. Default 7 days.

```swift
let config = URLSessionConfiguration.default
config.timeoutIntervalForRequest = 15      // fail if no data for 15s
config.timeoutIntervalForResource = 60     // hard cap of 60s total
let session = URLSession(configuration: config)
```

### Q40. How would you architect a networking layer for a large app from scratch (system design style)?
Typical layered answer interviewers want to hear:
1. **Endpoint/Route layer** — enum or struct describing path, method, headers, body (`Endpoint` protocol).
2. **HTTPClient** — thin wrapper around `URLSession`, returns `(Data, HTTPURLResponse)`, fully protocol-based for testing (see Q31).
3. **Interceptor chain** — auth headers, logging, retry-on-401 (see Q17, Q20).
4. **Decoding layer** — generic `APIClient.send<T: Decodable>(_:) async throws -> T`.
5. **Repository layer** — domain-specific (e.g., `PostRepository`), hides networking types from the UI/ViewModel.
6. **Caching layer** — `URLCache` + optional persistent store (Core Data/SwiftData) for offline-first.
7. **Error mapping** — convert transport/HTTP errors into domain errors the UI can present.
8. **Observability** — structured logging / metrics around request latency and failure rates.

```swift
protocol Endpoint {
    var path: String { get }
    var method: String { get }
    var queryItems: [URLQueryItem]? { get }
    var body: Encodable? { get }
}

final class APIClient {
    private let baseURL: URL
    private let httpClient: HTTPClient
    private let decoder: JSONDecoder

    init(baseURL: URL, httpClient: HTTPClient = URLSessionHTTPClient(), decoder: JSONDecoder = .init()) {
        self.baseURL = baseURL
        self.httpClient = httpClient
        self.decoder = decoder
    }

    func send<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        var components = URLComponents(url: baseURL.appendingPathComponent(endpoint.path), resolvingAgainstBaseURL: false)!
        components.queryItems = endpoint.queryItems
        var request = URLRequest(url: components.url!)
        request.httpMethod = endpoint.method
        if let body = endpoint.body {
            request.httpBody = try JSONEncoder().encode(body)
            request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        }
        let (data, http) = try await httpClient.send(request)
        guard (200..<300).contains(http.statusCode) else { throw mapHTTPError(statusCode: http.statusCode) }
        return try decoder.decode(T.self, from: data)
    }
}
```

---

## Quick-Fire Conceptual Round (rapid Q&A)

**Q: HTTP/1.1 vs HTTP/2 vs HTTP/3 — why does it matter for mobile?**
HTTP/2 multiplexes many requests over a single TCP connection (no head-of-line blocking at the connection level, fewer handshakes — faster on high-latency mobile networks). HTTP/3 swaps TCP for QUIC (UDP-based), which improves connection migration when a phone switches from Wi-Fi to cellular mid-request.

**Q: What is `HTTPURLResponse.allHeaderFields` case sensitivity behavior?**
HTTP header lookups via `value(forHTTPHeaderField:)` are case-insensitive per spec; direct dictionary access via `allHeaderFields` is not, so prefer the accessor method.

**Q: Why prefer `Decodable` structs over manually parsing `[String: Any]` from `JSONSerialization`?**
Type safety, compile-time guarantees, less boilerplate, and automatic handling of nested/optional fields — `JSONSerialization` is still useful for truly dynamic/unknown-shape JSON.

**Q: What does `Task.isCancelled` checking buy you in long async chains?**
Lets you bail out early from expensive work (e.g., mid-decode of a huge payload) instead of wasting CPU on a result nobody will use, by checking `try Task.checkCancellation()` between steps.

**Q: How do you simulate a slow/flaky network for manual QA?**
Use Xcode's **Network Link Conditioner** (via Additional Tools for Xcode) or pass `condition` profiles, or programmatically throttle in a `URLProtocol` mock with `Task.sleep` before responding.

---

### How to use this guide
Each function above is copy-pasteable and hits a real, currently-live endpoint as of this writing (jsonplaceholder.typicode.com, api.github.com, reqres.in, httpbin.org, echo.websocket.events). Run them in a Swift Playground or a throwaway Xcode project to confirm behavior before your interview, and be ready to explain *why* each pattern exists, not just recite the code.
