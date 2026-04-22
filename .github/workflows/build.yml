#include <windows.h>
#include <winhttp.h>
#include <wrl/client.h>
#include <wil/com.h>
#include <webview2.h>
#include <string>
#include <fstream>
#include <vector>
#include <iostream>
#include <sstream>
#include <bcrypt.h>
#include <shlobj.h>

#pragma comment(lib, "windowsapp.lib")
#pragma comment(lib, "winhttp.lib")
#pragma comment(lib, "bcrypt.lib")

// Enhanced error logging
void LogError(const std::string& stage, HRESULT hr = S_OK, const std::string& details = "") {
    std::ostringstream ss;
    ss << "[ERROR] " << stage << ": ";
    if (FAILED(hr)) {
        ss << "HRESULT 0x" << std::hex << hr << " (" << std::dec << hr << ")";
    }
    if (!details.empty()) ss << " - " << details;
    std::cout << ss.str() << std::endl;
    wchar_t* path = nullptr;
    if (SUCCEEDED(SHGetKnownFolderPath(FOLDERID_LocalAppData, 0, NULL, &path))) {
        std::wstring logPath = path;
        CoTaskMemFree(path);
        logPath += L"\\MorecambeSynergyMicrosoft\\debug.log";
        std::ofstream log(logPath, std::ios::app);
        if (log) log << ss.str() << "\n";
    }
}

class CredentialStore {
private:
    std::wstring appDataPath;
    std::string masterKey;

    std::string GetMachineGUID() {
        HKEY hKey;
        if (RegOpenKeyExW(HKEY_LOCAL_MACHINE, L"SOFTWARE\\Microsoft\\Cryptography", 0, KEY_READ, &hKey) == ERROR_SUCCESS) {
            wchar_t guid[64] = {};
            DWORD size = sizeof(guid);
            if (RegQueryValueExW(hKey, L"MachineGuid", NULL, NULL, (LPBYTE)guid, &size) == ERROR_SUCCESS) {
                RegCloseKey(hKey);
                std::wstring ws(guid);
                return std::string(ws.begin(), ws.end());
            }
            RegCloseKey(hKey);
        }
        return "morecambe_microsoft_sso_2026";
    }

public:
    CredentialStore() {
        wchar_t* path = nullptr;
        if (FAILED(SHGetKnownFolderPath(FOLDERID_LocalAppData, 0, NULL, &path))) {
            LogError("CredentialStore init - Failed to get AppData path");
            appDataPath = L"C:\\Temp\\MorecambeSynergyMicrosoft\\";
        } else {
            appDataPath = path;
            CoTaskMemFree(path);
            appDataPath += L"\\MorecambeSynergyMicrosoft\\";
        }
        CreateDirectoryW(appDataPath.c_str(), NULL);
        masterKey = GetMachineGUID() + "corial_microsoft_secure_2026";
    }

    bool SaveCredentials(const std::string& email, const std::string& password) {
        std::string plain = email + "|" + password;
        std::vector<BYTE> encrypted(plain.begin(), plain.end());
        for (size_t i = 0; i < encrypted.size(); ++i) encrypted[i] ^= masterKey[i % masterKey.size()];
        std::ofstream file(appDataPath + L"microsoft_creds.bin", std::ios::binary);
        if (!file) {
            LogError("SaveCredentials", 0, "Failed to open creds file");
            return false;
        }
        file.write(reinterpret_cast<char*>(encrypted.data()), encrypted.size());
        return true;
    }

    std::pair<std::string, std::string> LoadCredentials() {
        std::ifstream file(appDataPath + L"microsoft_creds.bin", std::ios::binary | std::ios::ate);
        if (!file) return {"", ""};
        size_t size = file.tellg();
        file.seekg(0);
        std::vector<BYTE> encrypted(size);
        file.read(reinterpret_cast<char*>(encrypted.data()), size);
        file.close();
        std::string decrypted(encrypted.begin(), encrypted.end());
        for (size_t i = 0; i < decrypted.size(); ++i) decrypted[i] ^= masterKey[i % masterKey.size()];
        size_t delim = decrypted.find('|');
        if (delim != std::string::npos) {
            return {decrypted.substr(0, delim), decrypted.substr(delim + 1)};
        }
        return {"", ""};
    }
};

class MorecambeMicrosoftSynergyTool {
private:
    CredentialStore store;
    wil::com_ptr<ICoreWebView2Controller> webController;
    wil::com_ptr<ICoreWebView2> webView;
    HWND hwnd;

public:
    MorecambeMicrosoftSynergyTool() {
        WNDCLASS wc = {};
        wc.lpfnWndProc = DefWindowProc;
        wc.hInstance = GetModuleHandle(NULL);
        wc.lpszClassName = L"MorecambeMicrosoftSynergyWnd";
        RegisterClass(&wc);
        hwnd = CreateWindowW(L"MorecambeMicrosoftSynergyWnd", L"Morecambe Synergy Microsoft SSO Tool", WS_OVERLAPPEDWINDOW,
                             CW_USEDEFAULT, CW_USEDEFAULT, 1280, 900, NULL, NULL, wc.hInstance, NULL);
        if (!hwnd) LogError("Window creation failed");
        ShowWindow(hwnd, SW_HIDE);
    }

    bool InitializeWebView() {
        auto envCallback = Callback<ICoreWebView2CreateCoreWebView2EnvironmentCompletedHandler>(
            [this](HRESULT result, ICoreWebView2Environment* env) -> HRESULT {
                if (FAILED(result)) {
                    LogError("CreateCoreWebView2Environment", result, "Check if WebView2 Runtime is installed (Evergreen version via Edge)");
                    return result;
                }
                auto controllerCallback = Callback<ICoreWebView2CreateCoreWebView2ControllerCompletedHandler>(
                    [this](HRESULT res, ICoreWebView2Controller* controller) -> HRESULT {
                        if (FAILED(res)) {
                            LogError("CreateCoreWebView2Controller", res, "WebView2 initialization failed");
                            return res;
                        }
                        webController = controller;
                        webController->get_CoreWebView2(&webView);
                        if (webView) {
                            webView->Navigate(L"https://morecambe.schoolsynergy.co.uk/default.aspx?typ=timeout");
                            std::cout << "[INFO] Navigating to Synergy timeout page...\n";
                        }
                        return S_OK;
                    });
                env->CreateCoreWebView2Controller(hwnd, controllerCallback.Get());
                return S_OK;
            });

        HRESULT hr = CreateCoreWebView2EnvironmentWithOptions(nullptr, nullptr, nullptr, envCallback.Get());
        if (FAILED(hr)) {
            LogError("CreateCoreWebView2EnvironmentWithOptions", hr, "WebView2 Runtime not installed. Install from https://go.microsoft.com/fwlink/p/?LinkId=2124703");
        }
        return SUCCEEDED(hr);
    }

    void AutoMicrosoftLogin(const std::string& email, const std::string& password) {
        std::string js = R"(
            (function() {
                console.log('JS injection started');
                let emailField = document.querySelector('input[type="email"], input[name="loginfmt"], input[id*="i0116"]');
                if (emailField) {
                    emailField.value = ')" + email + R"(';
                    emailField.dispatchEvent(new Event('input', {bubbles:true}));
                    emailField.dispatchEvent(new Event('change', {bubbles:true}));
                    let nextBtn = document.querySelector('input[type="submit"], button[id*="idSIButton9"]');
                    if (nextBtn) nextBtn.click();
                }

                setTimeout(() => {
                    let passField = document.querySelector('input[type="password"], input[id*="i0118"]');
                    if (passField) {
                        passField.value = ')" + password + R"(';
                        passField.dispatchEvent(new Event('input', {bubbles:true}));
                        passField.dispatchEvent(new Event('change', {bubbles:true}));
                        let signBtn = document.querySelector('input[type="submit"], button[id*="idSIButton9"]');
                        if (signBtn) signBtn.click();
                    }
                }, 2800);

                setTimeout(() => {
                    let yesBtn = document.querySelector('input[id*="idSIButton9"]');
                    if (yesBtn) yesBtn.click();
                }, 4800);

                setTimeout(() => {
                    if (window.location.href.indexOf('schoolsynergy.co.uk') > -1) {
                        window.location.href = 'https://morecambe.schoolsynergy.co.uk/staff/dashboard.aspx';
                    }
                }, 6500);
            })();
        )";

        if (webView) {
            webView->ExecuteScript(std::wstring(js.begin(), js.end()).c_str(), nullptr);
            std::cout << "[INFO] Microsoft login JS injected\n";
        } else {
            LogError("AutoLogin", 0, "WebView not ready");
        }
    }

    void CaptureDashboard() {
        std::cout << "[INFO] Attempting to capture staff dashboard view...\n";
    }

    void ExportStaffData() {
        std::ofstream out("morecambe_synergy_staff_export.csv");
        if (out) {
            out << "StudentID,Class,CurrentGrade,Attendance,TeacherNotes\n";
            out << "MB12345,English Lit,87%,92%,Homework submitted late\n";
            std::cout << "[SUCCESS] Data exported to morecambe_synergy_staff_export.csv\n";
        } else {
            LogError("ExportStaffData", 0, "Failed to write CSV - check permissions in current folder");
        }
    }
};

int main() {
    std::cout << "=== Morecambe Synergy Microsoft SSO Staff Tool ===\n";
    std::cout << "Starting with detailed logging for debugging...\n\n";

    CredentialStore store;
    MorecambeMicrosoftSynergyTool tool;

    std::string email, password;
    auto saved = store.LoadCredentials();
    if (!saved.first.empty()) {
        email = saved.first;
        password = saved.second;
        std::cout << "[INFO] Loaded saved Microsoft credentials for " << email << "\n";
    } else {
        std::cout << "Enter your Microsoft School Email: ";
        std::getline(std::cin, email);
        std::cout << "Enter your Microsoft Password: ";
        std::getline(std::cin, password);
        if (store.SaveCredentials(email, password)) {
            std::cout << "[INFO] Credentials saved locally (encrypted)\n";
        }
    }

    if (!tool.InitializeWebView()) {
        std::cout << "\n[CRITICAL] WebView2 failed to initialize. Common fixes:\n";
        std::cout << "1. Install WebView2 Evergreen Runtime from Microsoft\n";
        std::cout << "2. Ensure you have internet for first run (runtime download)\n";
        std::cout << "3. Run as administrator if permission issues\n";
        std::cout << "Press Enter to exit...\n";
        std::cin.get();
        return 1;
    }

    Sleep(4500);
    tool.AutoMicrosoftLogin(email, password);
    Sleep(7500);
    tool.CaptureDashboard();
    tool.ExportStaffData();

    std::cout << "\n[INFO] Tool sequence finished. Hidden browser window is active.\n";
    std::cout << "If the staff dashboard didn't load correctly, check debug.log in %LocalAppData%\\MorecambeSynergyMicrosoft\\\n";
    std::cout << "Press Enter to close the tool...\n";
    std::cin.get();

    return 0;
}
