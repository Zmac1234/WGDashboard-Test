# iOS App Prototype Setup

This document walks through creating a SwiftUI prototype that talks to a WGDashboard server. The server exposes REST endpoints like `handshake`, `authenticate`, and `isTotpEnabled` which the app can call. Example definitions from `src/dashboard.py`:

```python
@app.route(f'{APP_PREFIX}/api/handshake', methods=["GET", "OPTIONS"])
def API_Handshake():
    return ResponseObject(True)
```

```python
@app.get(f'{APP_PREFIX}/api/isTotpEnabled')
def API_isTotpEnabled():
    return (
        ResponseObject(data=DashboardConfig.GetConfig("Account", "enable_totp")[1] and DashboardConfig.GetConfig("Account", "totp_verified")[1]))
```

```python
@app.get(f'{APP_PREFIX}/api/getDashboardConfiguration')
def API_getDashboardConfiguration():
    cfg = DashboardConfig.toJson()
    _, port = DashboardConfig.GetConfig("Peers", "remote_endpoint_port")
    cfg["Peers"]["remote_endpoint_port"] = port
    return ResponseObject(data=cfg)
```

These endpoints let the app confirm the server is reachable, check if OTP is required, and determine if API keys are enabled.

## Project Setup

1. **Create a new project** in Xcode using the **App** template. Choose
   **SwiftUI** for the interface and **Swift** for the language.
2. Pick an appropriate **Product Name** (e.g. `WGDashboardClient`) and save
   the project to a convenient location.
3. In the Project Navigator you will see two files generated for you:
   `YourAppNameApp.swift` and `ContentView.swift`. These will be modified in the
   steps below.
4. For each Swift file mentioned later (`SessionManager.swift`,
   `NetworkManager.swift`, `ServerSetupView.swift`, `DashboardView.swift`), add a
   new file via **File → New → File…** and choose the **Swift File** template.

Once the files are in place you can copy the code snippets below into their
respective files.

## 1. SessionManager
Create or update `SessionManager.swift`:

```swift
import Foundation

class SessionManager: ObservableObject {
    @Published var isAuthenticated = false
    @Published var serverURL: URL? {
        didSet { UserDefaults.standard.set(serverURL?.absoluteString, forKey: "serverURL") }
    }
    @Published var apiKey: String? {
        didSet { UserDefaults.standard.set(apiKey, forKey: "apiKey") }
    }

    init() {
        if let str = UserDefaults.standard.string(forKey: "serverURL") {
            serverURL = URL(string: str)
        }
        apiKey = UserDefaults.standard.string(forKey: "apiKey")
    }
}
```

## 2. NetworkManager
Replace the static `baseURL` with a property that can be set at runtime. Add functions to fetch the dashboard configuration and OTP status.

```swift
import Foundation

final class NetworkManager {
    static let shared = NetworkManager()

    private var baseURL: URL?
    private var session: URLSession
    private init() {
        let config = URLSessionConfiguration.default
        config.httpCookieStorage = .shared
        session = URLSession(configuration: config)
    }

    func configure(baseURL: URL, apiKey: String?) {
        self.baseURL = baseURL
        if let key = apiKey {
            session.configuration.httpAdditionalHeaders = ["wg-dashboard-apikey": key]
        } else {
            session.configuration.httpAdditionalHeaders = nil
        }
    }

    func handshake(completion: @escaping (Bool) -> Void) {
        guard let url = baseURL?.appendingPathComponent("api/handshake") else { completion(false); return }
        session.dataTask(with: url) { data, _, _ in
            completion(data != nil)
        }.resume()
    }

    func fetchDashboardConfig(completion: @escaping ([String: Any]?) -> Void) {
        guard let url = baseURL?.appendingPathComponent("api/getDashboardConfiguration") else { completion(nil); return }
        session.dataTask(with: url) { data, _, _ in
            guard
                let data = data,
                let obj = try? JSONSerialization.jsonObject(with: data) as? [String: Any],
                let dataDict = obj["data"] as? [String: Any]
            else { completion(nil); return }
            completion(dataDict)
        }.resume()
    }

    func isTotpEnabled(completion: @escaping (Bool) -> Void) {
        guard let url = baseURL?.appendingPathComponent("api/isTotpEnabled") else { completion(false); return }
        session.dataTask(with: url) { data, _, _ in
            guard
                let data = data,
                let obj = try? JSONSerialization.jsonObject(with: data) as? [String: Any],
                let enabled = obj["data"] as? Bool
            else { completion(false); return }
            completion(enabled)
        }.resume()
    }

    func authenticate(username: String, password: String, totp: String?,
                      completion: @escaping (Bool, String?) -> Void) {
        guard let url = baseURL?.appendingPathComponent("api/authenticate") else {
            completion(false, "Bad URL"); return
        }
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        let body: [String: String] = [
            "username": username,
            "password": password,
            "totp": totp ?? ""
        ]
        request.httpBody = try? JSONEncoder().encode(body)

        session.dataTask(with: request) { data, _, _ in
            guard
                let data = data,
                let obj = try? JSONSerialization.jsonObject(with: data) as? [String: Any],
                let status = obj["status"] as? Bool
            else { completion(false, "Network error"); return }
            completion(status, obj["message"] as? String)
        }.resume()
    }

    func fetchConfigurations(completion: @escaping ([Configuration]?) -> Void) {
        guard let url = baseURL?.appendingPathComponent("api/getWireguardConfigurations") else { completion(nil); return }
        session.dataTask(with: url) { data, _, _ in
            guard
                let data = data,
                let obj = try? JSONSerialization.jsonObject(with: data) as? [String: Any],
                let status = obj["status"] as? Bool,
                status,
                let arr = obj["data"] as? [[String: Any]]
            else { completion(nil); return }

            if let json = try? JSONSerialization.data(withJSONObject: arr),
               let configs = try? JSONDecoder().decode([Configuration].self, from: json) {
                completion(configs)
            } else {
                completion(nil)
            }
        }.resume()
    }
}
```

## 3. ServerSetupView
Add a SwiftUI view that prompts for the server URL and (if required) an API key. Users may enter a full URL such as `https://example.com/prefix` or just a host/IP with a port like `192.168.2.43:10086`. If the input lacks a scheme, the view assumes `http://` automatically. Validate the address with `handshake` and `getDashboardConfiguration`.

```swift
import SwiftUI

struct ServerSetupView: View {
    @EnvironmentObject var session: SessionManager
    @State private var urlString = ""
    @State private var apiKey = ""
    @State private var requireApiKey = false
    @State private var error: String?

    var body: some View {
        Form {
            Section("Server") {
                TextField("https://example.com/prefix", text: $urlString)
                    .keyboardType(.URL)
                    .textInputAutocapitalization(.never)
            }
            if requireApiKey {
                Section("API Key") {
                    SecureField("API Key", text: $apiKey)
                }
            }
            if let err = error {
                Text(err).foregroundColor(.red)
            }
            Button("Continue", action: validate)
        }
        .navigationTitle("Connect")
    }

    private func makeURL(from text: String) -> URL? {
        if let url = URL(string: text), url.scheme != nil {
            return url
        }
        return URL(string: "http://\(text)")
    }

    private func validate() {
        guard let url = makeURL(from: urlString) else { error = "Invalid URL"; return }
        NetworkManager.shared.configure(baseURL: url, apiKey: nil)
        NetworkManager.shared.handshake { ok in
            guard ok else { DispatchQueue.main.async { self.error = "Server unreachable" }; return }
            NetworkManager.shared.fetchDashboardConfig { cfg in
                DispatchQueue.main.async {
                    guard let cfg = cfg else { self.error = "Config failed"; return }
                    if let server = cfg["Server"] as? [String: Any],
                       let apiKeyOn = server["dashboard_api_key"] as? String,
                       apiKeyOn == "true", !self.requireApiKey {
                        self.requireApiKey = true
                        self.error = "API key required"
                    } else {
                        NetworkManager.shared.configure(baseURL: url, apiKey: self.requireApiKey ? self.apiKey : nil)
                        session.serverURL = url
                        session.apiKey = self.requireApiKey ? self.apiKey : nil
                        session.isAuthenticated = false
                    }
                }
            }
        }
    }
}
```

## 4. ContentView.swift
Before showing the login form, check whether OTP is enabled by calling `isTotpEnabled`. This snippet includes the full sign‑in logic and `isSigningIn` flag so the button shows a progress view while the request is running.

```swift
struct ContentView: View {
    @EnvironmentObject var session: SessionManager
    @State private var username = ""
    @State private var password = ""
    @State private var otp = ""
    @State private var otpRequired = false
    @State private var isSigningIn = false
    @State private var errorText: String?

    var body: some View {
        NavigationStack {
            VStack(spacing: 16) {
                TextField("Username", text: $username)
                    .textFieldStyle(.roundedBorder)
                SecureField("Password", text: $password)
                    .textFieldStyle(.roundedBorder)
                if otpRequired {
                    TextField("OTP from your authenticator", text: $otp)
                        .textFieldStyle(.roundedBorder)
                        .keyboardType(.numberPad)
                }
                if let err = errorText {
                    Text(err).foregroundColor(.red)
                }
                Button(action: signIn) {
                    if isSigningIn {
                        ProgressView("Signing In…")
                    } else {
                        Label("Sign In", systemImage: "chevron.right")
                    }
                }
                .buttonStyle(.borderedProminent)
                .disabled(isSigningIn || username.isEmpty || password.isEmpty)
            }
            .padding()
            .navigationTitle("WGDashboard")
            .onAppear { NetworkManager.shared.isTotpEnabled { self.otpRequired = $0 } }
        }
    }

    private func signIn() {
        isSigningIn = true
        NetworkManager.shared.authenticate(
            username: username,
            password: password,
            totp: otpRequired ? otp : nil
        ) { success, message in
            DispatchQueue.main.async {
                self.isSigningIn = false
                if success {
                    session.isAuthenticated = true
                } else {
                    self.errorText = message ?? "Authentication failed"
                }
            }
        }
    }
}
```

## 5. App Entry Point
Update `WGDashboardClientApp.swift` to show `ServerSetupView` if `serverURL` is not yet saved.

```swift
@main
struct WGDashboardClientApp: App {
    @StateObject private var session = SessionManager()

    var body: some Scene {
        WindowGroup {
            if session.serverURL == nil {
                NavigationStack { ServerSetupView() }.environmentObject(session)
            } else if session.isAuthenticated {
                DashboardView().environmentObject(session)
            } else {
                ContentView().environmentObject(session)
            }
        }
    }
}
```

With these additions, the app first prompts for the server URL (and API key if required). It stores the values using `UserDefaults` so the prompt only appears on first launch. After validating the server, the login screen is shown. The OTP field only appears when `isTotpEnabled` returns `true`.

## Build and Run

1. Choose an iPhone simulator in the Xcode toolbar (for example **iPhone 15**).
2. Press **⌘R** or click the **Run** button to build the project.
3. When the app launches, enter the WGDashboard server URL. Provide the API key if the setup view indicates one is required.
4. After the server is validated the login form appears. Signing in will show the list of WireGuard configurations retrieved from your dashboard.
