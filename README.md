terminal injector kolay bir arayüzü var c++ yaptım direk visuala atın derle diyin bitti kullanmak için açıyorsun uygulamayı sonra cmd acıp tasklist yazıyorsun enter basıp hangi programa injectleyeceksen pidini buluyorsun mesela 4152 pidi cmdye yazıyorsun sonrada C:\Users\woize\Downloads\dutchlove2.dll böyle yazıp enter yapıyorsun oyuna injectleniyor eğer yapamadım diyorsanız https://discord.gg/5ZgDcN38m7 buraya gelip yardım alabilirsiniz







#include <windows.h>
#include <string>
#include <vector>
#include <iostream>
#include <tlhelp32.h>
#include "imgui.h"
#include "imgui_impl_win32.h"
#include "imgui_impl_dx9.h"
#include <d3d9.h>

#pragma comment(lib, "d3d9.lib")

LPDIRECT3D9              g_pD3D = nullptr;
LPDIRECT3DDEVICE9        g_pd3dDevice = nullptr;
HWND                     g_hwnd = nullptr;

bool CreateDeviceD3D(HWND hWnd) {
    if ((g_pD3D = Direct3DCreate9(D3D_SDK_VERSION)) == nullptr) return false;
    D3DPRESENT_PARAMETERS d3dpp = {};
    d3dpp.Windowed = TRUE;
    d3dpp.SwapEffect = D3DSWAPEFFECT_DISCARD;
    d3dpp.BackBufferFormat = D3DFMT_UNKNOWN;
    d3dpp.EnableAutoDepthStencil = TRUE;
    d3dpp.AutoDepthStencilFormat = D3DFMT_D16;
    if (g_pD3D->CreateDevice(D3DADAPTER_DEFAULT, D3DDEVTYPE_HAL, hWnd,
        D3DCREATE_HARDWARE_VERTEXPROCESSING, &d3dpp, &g_pd3dDevice) < 0)
        return false;
    return true;
}

void CleanupDeviceD3D() {
    if (g_pd3dDevice) { g_pd3dDevice->Release(); g_pd3dDevice = nullptr; }
    if (g_pD3D) { g_pD3D->Release(); g_pD3D = nullptr; }
}

DWORD GetProcessID(const std::string& processName) {
    DWORD pid = 0;
    PROCESSENTRY32 pe;
    pe.dwSize = sizeof(PROCESSENTRY32);
    HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);

    if (Process32First(snapshot, &pe)) {
        do {
            if (_stricmp(pe.szExeFile, processName.c_str()) == 0) {
                pid = pe.th32ProcessID;
                break;
            }
        } while (Process32Next(snapshot, &pe));
    }
    CloseHandle(snapshot);
    return pid;
}

bool InjectDLL(DWORD pid, const std::string& dllPath) {
    HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, pid);
    if (!hProcess) return false;

    void* allocatedMem = VirtualAllocEx(hProcess, nullptr, dllPath.size(), MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    if (!allocatedMem) return false;

    WriteProcessMemory(hProcess, allocatedMem, dllPath.c_str(), dllPath.size(), nullptr);
    HANDLE hThread = CreateRemoteThread(hProcess, nullptr, 0, (LPTHREAD_START_ROUTINE)GetProcAddress(GetModuleHandle("kernel32.dll"), "LoadLibraryA"), allocatedMem, 0, nullptr);
    if (!hThread) return false;

    WaitForSingleObject(hThread, INFINITE);
    VirtualFreeEx(hProcess, allocatedMem, 0, MEM_RELEASE);
    CloseHandle(hThread);
    CloseHandle(hProcess);

    return true;
}

LRESULT CALLBACK WndProc(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam) {
    if (ImGui_ImplWin32_WndProcHandler(hWnd, msg, wParam, lParam)) return true;
    return DefWindowProc(hWnd, msg, wParam, lParam);
}

int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE, LPSTR, int) {
    WNDCLASSEX wc = { sizeof(WNDCLASSEX), CS_CLASSDC, WndProc, 0L, 0L, GetModuleHandle(NULL), NULL, NULL, NULL, NULL, "DLLInjector", NULL };
    RegisterClassEx(&wc);
    g_hwnd = CreateWindow(wc.lpszClassName, "Mage's DLL Injector", WS_OVERLAPPEDWINDOW, 100, 100, 400, 200, NULL, NULL, wc.hInstance, NULL);
    
    if (!CreateDeviceD3D(g_hwnd)) {
        CleanupDeviceD3D();
        UnregisterClass(wc.lpszClassName, wc.hInstance);
        return 0;
    }

    ShowWindow(g_hwnd, SW_SHOWDEFAULT);
    UpdateWindow(g_hwnd);
    
    ImGui::CreateContext();
    ImGuiIO& io = ImGui::GetIO(); (void)io;
    ImGui::StyleColorsDark();
    ImGui_ImplWin32_Init(g_hwnd);
    ImGui_ImplDX9_Init(g_pd3dDevice);

    bool show_app = true;
    std::string dllPath;
    char exeName[128] = "";
    
    while (show_app) {
        MSG msg;
        while (PeekMessage(&msg, NULL, 0U, 0U, PM_REMOVE)) {
            TranslateMessage(&msg);
            DispatchMessage(&msg);
            if (msg.message == WM_QUIT) show_app = false;
        }

        ImGui_ImplDX9_NewFrame();
        ImGui_ImplWin32_NewFrame();
        ImGui::NewFrame();

        ImGui::Begin("Mage's DLL Injector", &show_app);
        ImGui::Text("Choose Running Application (.exe):");
        ImGui::InputText("##exe", exeName, IM_ARRAYSIZE(exeName));
        ImGui::Text("DLL Path:");
        if (ImGui::Button("Load DLL")) {
            OPENFILENAME ofn;
            char szFile[260] = { 0 };
            ZeroMemory(&ofn, sizeof(ofn));
            ofn.lStructSize = sizeof(ofn);
            ofn.hwndOwner = g_hwnd;
            ofn.lpstrFilter = "DLL Files\0*.dll\0";
            ofn.lpstrFile = szFile;
            ofn.nMaxFile = sizeof(szFile);
            ofn.Flags = OFN_FILEMUSTEXIST;
            if (GetOpenFileName(&ofn)) dllPath = szFile;
        }
        ImGui::Text("Status: Waiting for injection...");
        if (ImGui::Button("Inject")) {
            DWORD pid = GetProcessID(exeName);
            if (pid) {
                if (InjectDLL(pid, dllPath)) {
                    ImGui::Text("DLL injected successfully!");
                } else {
                    ImGui::Text("Injection failed!");
                }
            } else {
                ImGui::Text("Process not found!");
            }
        }
        ImGui::End();

        ImGui::Render();
        g_pd3dDevice->SetRenderState(D3DRS_ZENABLE, FALSE);
        g_pd3dDevice->SetRenderState(D3DRS_ALPHABLENDENABLE, FALSE);
        g_pd3dDevice->SetRenderState(D3DRS_SCISSORTESTENABLE, FALSE);
        g_pd3dDevice->Clear(0, NULL, D3DCLEAR_TARGET, D3DCOLOR_RGBA(0, 0, 0, 255), 1.0f, 0);
        if (g_pd3dDevice->BeginScene() >= 0) {
            ImGui::Render();
            ImGui_ImplDX9_RenderDrawData(ImGui::GetDrawData());
            g_pd3dDevice->EndScene();
        }
        g_pd3dDevice->Present(NULL, NULL, NULL, NULL);
    }
    
    CleanupDeviceD3D();
    return 0;
}
