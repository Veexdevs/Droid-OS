mkdir -p ~/droidos/src ~/droidos/assets ~/droidos/third_party ~/droidos/build
cd ~/droidos || exit 1

# pacotes (só instala se faltar)
command -v g++ >/dev/null 2>&1 || pkg install -y clang
command -v curl >/dev/null 2>&1 || pkg install -y curl
if [ ! -f "$PREFIX/include/X11/Xlib.h" ] || [ ! -f "$PREFIX/include/EGL/egl.h" ]; then
  pkg install -y x11-repo
  pkg install -y libx11 xorgproto mesa
fi
command -v termux-x11 >/dev/null 2>&1 || pkg install -y termux-x11-nightly
command -v termux-volume >/dev/null 2>&1 || pkg install -y termux-api

# baixa um arquivo (pula se já existe; desiste em 40s; descarta página de erro)
dl() {
  [ -s "$2" ] && return 0
  echo "  baixando $2 ..."
  curl -fsSL --connect-timeout 10 --max-time 40 -A "Mozilla/5.0" -o "$2" "$1" </dev/null || { rm -f "$2"; return 1; }
  [ "$(head -c 1 "$2")" = "<" ] && rm -f "$2"
  [ -s "$2" ]
}

echo "==> Baixando arquivos"
dl https://raw.githubusercontent.com/nothings/stb/master/stb_truetype.h third_party/stb_truetype.h
dl https://raw.githubusercontent.com/nothings/stb/master/stb_image.h third_party/stb_image.h
dl "https://i.postimg.cc/59PJkTpD/143-Sem-Titulo-20260530003251.png" assets/logo.png
dl "https://commons.wikimedia.org/wiki/Special:FilePath/Firefox_logo%2C_2019.svg?width=512" assets/firefox.png
if [ ! -s assets/firefox.png ]; then
  for f in "$PREFIX"/share/icons/hicolor/*/apps/firefox.png; do [ -f "$f" ] && cp "$f" assets/firefox.png; done
fi
ls /system/fonts/Roboto* >/dev/null 2>&1 || dl "https://github.com/dejavu-fonts/dejavu-fonts/raw/version_2_37/ttf/DejaVuSans.ttf" assets/font.ttf

if [ ! -s third_party/stb_truetype.h ] || [ ! -s third_party/stb_image.h ]; then
  echo "ERRO: não consegui baixar os arquivos stb (sem internet?). Rode o bloco de novo."
  exit 1
fi
[ -s assets/logo.png ]    || echo "Aviso: a logo não baixou. Copie sua imagem para ~/droidos/assets/logo.png (por enquanto aparece o robô)."
[ -s assets/firefox.png ] || echo "Aviso: o ícone do Firefox não baixou (uso um desenho simples)."

# ---- código do DroidOS ----
cat > src/main.cpp << 'DROIDOS_MAIN_EOF'
// ============================================================
//  DroidOS - desktop para Termux X11 (X11 + EGL + GLES3)
//  third_party/: stb_truetype.h, stb_image.h  (baixados pelo install.sh)
//  assets/: logo.png (logo do personagem), firefox.png, font.ttf (opcional)
//  A pasta do projeto vem da variável DROIDOS_DIR (o run.sh define).
// ============================================================
#define STB_TRUETYPE_IMPLEMENTATION
#include "stb_truetype.h"
#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <math.h>
#include <time.h>
#include <unistd.h>
#include <dirent.h>
#include <pthread.h>
#include <X11/Xlib.h>
#include <X11/Xutil.h>
#include <X11/keysym.h>
#include <EGL/egl.h>
#include <GLES3/gl3.h>

// ---------- Shaders (mesmo esquema de antes: aPos/aColor + uScale/uOffset; agora com textura) ----------
const char* vertexShaderSource = R"(#version 300 es
layout (location = 0) in vec2 aPos;
layout (location = 1) in vec4 aColor;
layout (location = 2) in vec2 aUV;

uniform vec2 uScale;
uniform vec2 uOffset;

out vec4 vertexColor;
out vec2 vUV;

void main() {
    vec2 scaledPos = (aPos * uScale) + uOffset;
    gl_Position = vec4(scaledPos, 0.0, 1.0);
    vertexColor = aColor;
    vUV = aUV;
}
)";

const char* fragmentShaderSource = R"(#version 300 es
precision mediump float;

uniform sampler2D uTex;
in vec4 vertexColor;
in vec2 vUV;
out vec4 FragColor;

void main() {
    FragColor = vertexColor * texture(uTex, vUV);
}
)";

// ============================================================
//  Tipos e cores
// ============================================================
struct Color { float r, g, b, a; };
struct Rect  { float x, y, w, h; };

static inline Color C(float r, float g, float b, float a = 1.0f) { Color c = {r, g, b, a}; return c; }

static const Color WALLPAPER = {1.00f, 1.00f, 1.00f, 1.0f};
static const Color TASKBAR   = {0.00f, 0.00f, 0.00f, 1.0f};
static const Color WINDOW_BG = {0.25f, 0.25f, 0.25f, 1.0f};
static const Color WHITE     = {1.00f, 1.00f, 1.00f, 1.0f};
static const Color DIM       = {1.00f, 1.00f, 1.00f, 0.30f};
static const Color GRAYTXT   = {0.72f, 0.72f, 0.76f, 1.0f};
static const Color POPUP_BG  = {0.12f, 0.12f, 0.14f, 0.98f};
static const Color ROW_BG    = {0.20f, 0.20f, 0.23f, 1.0f};
static const Color ACCENT    = {0.00f, 0.47f, 0.84f, 1.0f};

static inline bool inRect(const Rect& r, float px, float py) {
    return px >= r.x && px <= r.x + r.w && py >= r.y && py <= r.y + r.h;
}

static long nowMs() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return (long)ts.tv_sec * 1000L + ts.tv_nsec / 1000000L;
}

// ============================================================
//  Atlas de textura: pixel branco + fontes + ícones
// ============================================================
#define ATLAS_W 1024
#define ATLAS_H 1536
#define WHITE_U (4.0f / ATLAS_W)
#define WHITE_V (4.0f / ATLAS_H)

static unsigned char* g_atlas = NULL;

struct Icon { bool ok; float u0, v0, u1, v1; };
static Icon g_iconLogo = {false, 0, 0, 0, 0};
static Icon g_iconFox  = {false, 0, 0, 0, 0};

static unsigned char* readFile(const char* path) {
    FILE* f = fopen(path, "rb");
    if (!f) return NULL;
    fseek(f, 0, SEEK_END);
    long n = ftell(f);
    fseek(f, 0, SEEK_SET);
    if (n <= 0) { fclose(f); return NULL; }
    unsigned char* b = (unsigned char*)malloc((size_t)n);
    if (!b) { fclose(f); return NULL; }
    size_t rd = fread(b, 1, (size_t)n, f);
    fclose(f);
    if ((long)rd != n) { free(b); return NULL; }
    return b;
}

// procura um arquivo em <DROIDOS_DIR>/assets, <DROIDOS_DIR> e ~/test_local
static bool findAsset(const char* name, char* out, size_t n) {
    const char* dir = getenv("DROIDOS_DIR");
    const char* home = getenv("HOME");
    if (!home) home = "";
    char base[500];
    if (dir && dir[0]) snprintf(base, sizeof(base), "%s", dir);
    else snprintf(base, sizeof(base), "%s/droidos", home);
    snprintf(out, n, "%s/assets/%s", base, name);
    if (access(out, R_OK) == 0) return true;
    snprintf(out, n, "%s/%s", base, name);
    if (access(out, R_OK) == 0) return true;
    snprintf(out, n, "%s/test_local/%s", home, name);
    return access(out, R_OK) == 0;
}

// carrega PNG/JPG, redimensiona (média ponderada por alpha) e cola numa região size x size do atlas
static bool loadIconTo(const char* path, int dx0, int dy0, int size, Icon* ic) {
    int w = 0, h = 0, n = 0;
    unsigned char* src = stbi_load(path, &w, &h, &n, 4);
    if (!src || w < 1 || h < 1) return false;

    float sc = fminf((float)size / w, (float)size / h);
    int dw = (int)(w * sc + 0.5f), dh = (int)(h * sc + 0.5f);
    if (dw < 1) dw = 1;
    if (dh < 1) dh = 1;
    if (dw > size) dw = size;
    if (dh > size) dh = size;
    int ox = (size - dw) / 2, oy = (size - dh) / 2;

    for (int y = 0; y < dh; y++) {
        for (int x = 0; x < dw; x++) {
            int sx0 = (int)floorf(x / sc), sx1 = (int)ceilf((x + 1) / sc);
            int sy0 = (int)floorf(y / sc), sy1 = (int)ceilf((y + 1) / sc);
            if (sx1 <= sx0) sx1 = sx0 + 1;
            if (sy1 <= sy0) sy1 = sy0 + 1;
            if (sx1 > w) sx1 = w;
            if (sy1 > h) sy1 = h;
            if (sx0 >= sx1) sx0 = sx1 - 1;
            if (sy0 >= sy1) sy0 = sy1 - 1;

            double ar = 0, ag = 0, ab = 0, aa = 0;
            int cnt = 0;
            for (int sy = sy0; sy < sy1; sy++) {
                for (int sx = sx0; sx < sx1; sx++) {
                    const unsigned char* p = src + ((size_t)sy * w + sx) * 4;
                    double a = p[3];
                    ar += p[0] * a; ag += p[1] * a; ab += p[2] * a; aa += a;
                    cnt++;
                }
            }
            unsigned char* d = g_atlas + ((size_t)(dy0 + oy + y) * ATLAS_W + (dx0 + ox + x)) * 4;
            if (aa > 0 && cnt > 0) {
                d[0] = (unsigned char)(ar / aa);
                d[1] = (unsigned char)(ag / aa);
                d[2] = (unsigned char)(ab / aa);
                d[3] = (unsigned char)(aa / cnt);
            } else {
                d[0] = d[1] = d[2] = d[3] = 0;
            }
        }
    }
    stbi_image_free(src);

    ic->ok = true;
    ic->u0 = (dx0 + 1.0f) / ATLAS_W;
    ic->v0 = (dy0 + 1.0f) / ATLAS_H;
    ic->u1 = (dx0 + size - 1.0f) / ATLAS_W;
    ic->v1 = (dy0 + size - 1.0f) / ATLAS_H;
    return true;
}

// ============================================================
//  Fontes normais (TrueType) -> atlas
// ============================================================
static const float BAKE_PX[2] = {22.0f, 36.0f};
static stbtt_bakedchar g_cd[2][2][224];   // [negrito][tamanho][char-32]
static bool g_fontOK = false;
static bool g_fauxBold = false;

static bool hasBad(const char* nm) {
    static const char* bad[] = {"Italic", "Mono", "Serif", "Slab", "Condensed", "Flex", "CJK", "Emoji", "Symbol", NULL};
    for (int i = 0; bad[i]; i++) if (strstr(nm, bad[i])) return true;
    return false;
}

static unsigned char* loadFontFile(bool bold) {
    char p[600];
    const char* prefix = getenv("PREFIX"); if (!prefix) prefix = "";
    unsigned char* b;

    if (findAsset(bold ? "font-bold.ttf" : "font.ttf", p, sizeof(p)) && (b = readFile(p))) return b;

    static const char* regList[] = {
        "/system/fonts/Roboto-Regular.ttf", "/system/fonts/RobotoStatic-Regular.ttf",
        "/system/fonts/GoogleSans-Regular.ttf", "/system/fonts/GoogleSansText-Regular.ttf",
        "/system/fonts/OpenSans-Regular.ttf", "/system/fonts/NotoSans-Regular.ttf",
        "/system/fonts/DroidSans.ttf", NULL};
    static const char* boldList[] = {
        "/system/fonts/Roboto-Bold.ttf", "/system/fonts/RobotoStatic-Bold.ttf",
        "/system/fonts/GoogleSans-Bold.ttf", "/system/fonts/GoogleSansText-Bold.ttf",
        "/system/fonts/OpenSans-Bold.ttf", "/system/fonts/NotoSans-Bold.ttf",
        "/system/fonts/DroidSans-Bold.ttf", NULL};
    const char** list = bold ? boldList : regList;
    for (int i = 0; list[i]; i++) if ((b = readFile(list[i]))) return b;

    snprintf(p, sizeof(p), "%s/share/fonts/TTF/%s", prefix, bold ? "DejaVuSans-Bold.ttf" : "DejaVuSans.ttf");
    if ((b = readFile(p))) return b;

    // varredura em /system/fonts
    DIR* d = opendir("/system/fonts");
    if (d) {
        static const char* pref[] = {"Roboto", "GoogleSans", "OpenSans", "SamsungOne", "Inter", NULL};
        struct dirent* e;
        while ((e = readdir(d))) {
            const char* nm = e->d_name;
            size_t L = strlen(nm);
            if (L < 5 || (strcmp(nm + L - 4, ".ttf") && strcmp(nm + L - 4, ".otf"))) continue;
            bool okPref = false;
            for (int i = 0; pref[i]; i++) if (strncmp(nm, pref[i], strlen(pref[i])) == 0) okPref = true;
            if (!okPref || hasBad(nm)) continue;
            bool isBold = strstr(nm, "Bold") != NULL;
            bool isReg = strstr(nm, "Regular") != NULL || strchr(nm, '[') != NULL;
            if ((bold && isBold) || (!bold && isReg && !isBold)) {
                snprintf(p, sizeof(p), "/system/fonts/%s", nm);
                if ((b = readFile(p))) { closedir(d); return b; }
            }
        }
        closedir(d);
    }
    return NULL;
}

static bool bakeInto(const unsigned char* ttf, int bold) {
    int off = stbtt_GetFontOffsetForIndex(ttf, 0);
    for (int s = 0; s < 2; s++) {
        unsigned char* tmp = (unsigned char*)calloc(512 * 512, 1);
        if (!tmp) return false;
        int r = stbtt_BakeFontBitmap(ttf, off, BAKE_PX[s], tmp, 512, 512, 32, 224, g_cd[bold][s]);
        if (r <= 0) { free(tmp); return false; }
        int ox = bold ? 512 : 0, oy = 16 + s * 512;
        for (int y = 0; y < 512; y++) {
            for (int x = 0; x < 512; x++) {
                unsigned char* d = g_atlas + ((size_t)(oy + y) * ATLAS_W + ox + x) * 4;
                d[0] = d[1] = d[2] = 255;
                d[3] = tmp[y * 512 + x];
            }
        }
        free(tmp);
    }
    return true;
}

static void initFonts() {
    unsigned char* reg = loadFontFile(false);
    if (!reg) { fprintf(stderr, "[DroidOS] nenhuma fonte encontrada (coloque font.ttf em assets/)\n"); return; }
    unsigned char* bld = loadFontFile(true);
    if (!bld) { bld = reg; g_fauxBold = true; }
    if (!bakeInto(reg, 0)) { fprintf(stderr, "[DroidOS] falha ao carregar a fonte\n"); return; }
    if (!bakeInto(bld, 1)) { g_fauxBold = true; bakeInto(reg, 1); }
    g_fontOK = true;
}

// ============================================================
//  Buffer de vértices (x, y, r, g, b, a, u, v) montado na CPU
// ============================================================
#define MAX_FLOATS 800000
static float g_buf[MAX_FLOATS];
static int   g_count = 0;
static float g_W = 1.0f, g_H = 1.0f;
static const float PI_F = 3.14159265f;

static void pushVert(float px, float py, Color c, float u, float v) {
    if (g_count + 8 > MAX_FLOATS) return;
    g_buf[g_count++] = px / g_W * 2.0f - 1.0f;
    g_buf[g_count++] = 1.0f - py / g_H * 2.0f;
    g_buf[g_count++] = c.r; g_buf[g_count++] = c.g; g_buf[g_count++] = c.b; g_buf[g_count++] = c.a;
    g_buf[g_count++] = u;   g_buf[g_count++] = v;
}

static void tri(float x1, float y1, float x2, float y2, float x3, float y3, Color c) {
    pushVert(x1, y1, c, WHITE_U, WHITE_V);
    pushVert(x2, y2, c, WHITE_U, WHITE_V);
    pushVert(x3, y3, c, WHITE_U, WHITE_V);
}

static void rect(float x, float y, float w, float h, Color c) {
    tri(x, y, x, y + h, x + w, y + h, c);
    tri(x, y, x + w, y + h, x + w, y, c);
}

static void quadUV(float x, float y, float w, float h, float u0, float v0, float u1, float v1, Color c) {
    pushVert(x, y, c, u0, v0);         pushVert(x, y + h, c, u0, v1);     pushVert(x + w, y + h, c, u1, v1);
    pushVert(x, y, c, u0, v0);         pushVert(x + w, y + h, c, u1, v1); pushVert(x + w, y, c, u1, v0);
}

static void fan(float cx, float cy, float r, float a0, float a1, int segs, Color c) {
    for (int i = 0; i < segs; i++) {
        float aa = a0 + (a1 - a0) * i / segs;
        float ab = a0 + (a1 - a0) * (i + 1) / segs;
        tri(cx, cy, cx + r * cosf(aa), cy + r * sinf(aa), cx + r * cosf(ab), cy + r * sinf(ab), c);
    }
}

static void circle(float cx, float cy, float r, Color c) {
    int segs = r < 6 ? 16 : (r < 20 ? 28 : 44);
    fan(cx, cy, r, 0.0f, 2.0f * PI_F, segs, c);
}

static void rrect(float x, float y, float w, float h, float r, Color c) {
    r = fminf(r, fminf(w, h) * 0.5f);
    if (r < 1.0f) { rect(x, y, w, h, c); return; }
    rect(x + r, y, w - 2 * r, h, c);
    rect(x, y + r, r, h - 2 * r, c);
    rect(x + w - r, y + r, r, h - 2 * r, c);
    fan(x + r,     y + r,     r, PI_F,        1.5f * PI_F, 10, c);
    fan(x + w - r, y + r,     r, 1.5f * PI_F, 2.0f * PI_F, 10, c);
    fan(x + w - r, y + h - r, r, 0.0f,        0.5f * PI_F, 10, c);
    fan(x + r,     y + h - r, r, 0.5f * PI_F, PI_F,        10, c);
}

static void capsule(float ax, float ay, float bx, float by, float rad, Color c) {
    float dx = bx - ax, dy = by - ay;
    float len = sqrtf(dx * dx + dy * dy);
    if (len < 0.001f) { circle(ax, ay, rad, c); return; }
    float nx = -dy / len * rad, ny = dx / len * rad;
    tri(ax + nx, ay + ny, ax - nx, ay - ny, bx - nx, by - ny, c);
    tri(ax + nx, ay + ny, bx - nx, by - ny, bx + nx, by + ny, c);
    circle(ax, ay, rad, c);
    circle(bx, by, rad, c);
}

static void ringSector(float cx, float cy, float r0, float r1, float a0, float a1, int segs, Color c) {
    for (int i = 0; i < segs; i++) {
        float aa = a0 + (a1 - a0) * i / segs;
        float ab = a0 + (a1 - a0) * (i + 1) / segs;
        float x1 = cx + r0 * cosf(aa), y1 = cy + r0 * sinf(aa);
        float x2 = cx + r1 * cosf(aa), y2 = cy + r1 * sinf(aa);
        float x3 = cx + r1 * cosf(ab), y3 = cy + r1 * sinf(ab);
        float x4 = cx + r0 * cosf(ab), y4 = cy + r0 * sinf(ab);
        tri(x1, y1, x2, y2, x3, y3, c);
        tri(x1, y1, x3, y3, x4, y4, c);
    }
}

static void drawIcon(const Icon& ic, float x, float y, float w, float h, Color tint) {
    quadUV(x, y, w, h, ic.u0, ic.v0, ic.u1, ic.v1, tint);
}

// ============================================================
//  Texto (UTF-8 -> Latin-1)
// ============================================================
static int nextCP(const char** ps) {
    const unsigned char* s = (const unsigned char*)*ps;
    int cp = s[0];
    if (cp < 0x80) { *ps += 1; return cp; }
    if ((cp & 0xE0) == 0xC0 && (s[1] & 0xC0) == 0x80) { *ps += 2; return ((cp & 0x1F) << 6) | (s[1] & 0x3F); }
    if ((cp & 0xF0) == 0xE0) { *ps += 3; return '?'; }
    if ((cp & 0xF8) == 0xF0) { *ps += 4; return '?'; }
    *ps += 1;
    return '?';
}

static int sizeIdx(float size) { return size <= 26.0f ? 0 : 1; }

static float textWidth(const char* s, float size, bool bold) {
    if (!g_fontOK) return 0;
    int si = sizeIdx(size);
    float sc = size / BAKE_PX[si], w = 0;
    while (*s) {
        int cp = nextCP(&s);
        if (cp < 32 || cp > 255) cp = '?';
        w += g_cd[bold ? 1 : 0][si][cp - 32].xadvance * sc;
    }
    return w;
}

// x = início do texto, cy = centro vertical
static void drawText(const char* s, float x, float cy, float size, bool bold, Color c) {
    if (!g_fontOK) return;
    int si = sizeIdx(size), bi = bold ? 1 : 0;
    float sc = size / BAKE_PX[si];
    float base = cy + size * 0.36f;
    int passes = (bold && g_fauxBold) ? 2 : 1;
    for (int pass = 0; pass < passes; pass++) {
        const char* p = s;
        float px = x + pass * size * 0.035f;
        while (*p) {
            int cp = nextCP(&p);
            if (cp < 32 || cp > 255) cp = '?';
            const stbtt_bakedchar& b = g_cd[bi][si][cp - 32];
            int w = b.x1 - b.x0, h = b.y1 - b.y0;
            if (w > 0 && h > 0) {
                float gx = floorf(px + b.xoff * sc + 0.5f);
                float gy = floorf(base + b.yoff * sc + 0.5f);
                int ox = bi ? 512 : 0, oy = 16 + si * 512;
                quadUV(gx, gy, w * sc, h * sc,
                       (ox + b.x0) / (float)ATLAS_W, (oy + b.y0) / (float)ATLAS_H,
                       (ox + b.x1) / (float)ATLAS_W, (oy + b.y1) / (float)ATLAS_H, c);
            }
            px += b.xadvance * sc;
        }
    }
}

static void drawTextRight(const char* s, float right, float cy, float size, bool bold, Color c) {
    drawText(s, right - textWidth(s, size, bold), cy, size, bold, c);
}
static void drawTextCenter(const char* s, float cx, float cy, float size, bool bold, Color c) {
    drawText(s, cx - textWidth(s, size, bold) * 0.5f, cy, size, bold, c);
}

// ============================================================
//  Sistema (Wi-Fi, dados móveis, volume, brilho) via Termux:API
// ============================================================
struct SysInfo {
    int  wifiLevel;      // -1 desconectado, 0..3
    int  rssi;
    char ssid[64];
    int  mobLevel;       // 0..4
    char netGen[8];      // "2G" "3G" "4G" "5G"
    char oper[64];
    int  volPct, volMax; // 0..100 / maximo do stream music
    int  briPct;         // 0..100
};

static SysInfo g_sys = {3, -55, "", 4, "4G", "", 50, 15, 60};
static pthread_mutex_t g_mu = PTHREAD_MUTEX_INITIALIZER;
static long g_userTouchVol = 0;

static bool runCmd(const char* cmd, char* out, size_t n) {
    FILE* f = popen(cmd, "r");
    if (!f) return false;
    size_t len = 0, rd;
    while (len + 1 < n && (rd = fread(out + len, 1, n - 1 - len, f)) > 0) len += rd;
    out[len] = 0;
    pclose(f);
    return len > 0;
}

static bool jsonNumber(const char* s, const char* key, double* out) {
    char pat[80];
    snprintf(pat, sizeof(pat), "\"%s\"", key);
    const char* p = strstr(s, pat);
    if (!p) return false;
    p += strlen(pat);
    while (*p == ' ' || *p == ':' || *p == '\t' || *p == '\n' || *p == '\r') p++;
    char* end;
    double v = strtod(p, &end);
    if (end == p) return false;
    *out = v;
    return true;
}

static bool jsonString(const char* s, const char* key, char* out, size_t n) {
    char pat[80];
    snprintf(pat, sizeof(pat), "\"%s\"", key);
    const char* p = strstr(s, pat);
    if (!p) return false;
    p += strlen(pat);
    while (*p == ' ' || *p == ':' || *p == '\t' || *p == '\n' || *p == '\r') p++;
    if (*p != '"') return false;
    p++;
    size_t i = 0;
    while (*p && *p != '"' && i + 1 < n) {
        if (*p == '\\' && p[1]) p++;
        out[i++] = *p++;
    }
    out[i] = 0;
    return true;
}

static void netGenFrom(const char* type, char* out) {
    char t[48];
    size_t i = 0;
    for (; type[i] && i < sizeof(t) - 1; i++) t[i] = (type[i] >= 'A' && type[i] <= 'Z') ? type[i] + 32 : type[i];
    t[i] = 0;
    const char* g = "4G";
    if (strcmp(t, "nr") == 0 || strstr(t, "5g")) g = "5G";
    else if (strstr(t, "lte")) g = "4G";
    else if (strstr(t, "hsdpa") || strstr(t, "hsupa") || strstr(t, "hspa") || strstr(t, "umts") ||
             strstr(t, "utran") || strstr(t, "scdma") || strstr(t, "evdo") || strstr(t, "ehrpd")) g = "3G";
    else if (strstr(t, "edge") || strstr(t, "gprs") || strstr(t, "gsm") || strstr(t, "cdma") ||
             strstr(t, "1xrtt") || strstr(t, "iden")) g = "2G";
    else if (t[0] == 0 || strstr(t, "unknown")) g = "4G";
    strcpy(out, g);
}

static void pollOnce(int tick) {
    static char buf[16384];
    double v;
    char tmp[128];

    if (runCmd("timeout 6 termux-wifi-connectioninfo 2>/dev/null", buf, sizeof(buf))) {
        int rssi = -127;
        char st[40] = "", ssid[64] = "";
        if (jsonNumber(buf, "rssi", &v)) rssi = (int)v;
        jsonString(buf, "supplicant_state", st, sizeof(st));
        jsonString(buf, "ssid", ssid, sizeof(ssid));
        bool conn = (strcmp(st, "COMPLETED") == 0) || (st[0] == 0 && rssi > -120);
        int lvl = -1;
        if (conn) lvl = rssi >= -60 ? 3 : (rssi >= -72 ? 2 : (rssi >= -85 ? 1 : 0));
        pthread_mutex_lock(&g_mu);
        g_sys.wifiLevel = lvl; g_sys.rssi = rssi;
        snprintf(g_sys.ssid, sizeof(g_sys.ssid), "%s", ssid);
        pthread_mutex_unlock(&g_mu);
    }

    if (runCmd("timeout 6 termux-telephony-deviceinfo 2>/dev/null", buf, sizeof(buf))) {
        char type[48] = "", oper[64] = "", gen[8];
        jsonString(buf, "data_network_type", type, sizeof(type));
        jsonString(buf, "network_operator_name", oper, sizeof(oper));
        netGenFrom(type, gen);
        pthread_mutex_lock(&g_mu);
        snprintf(g_sys.netGen, sizeof(g_sys.netGen), "%s", gen);
        snprintf(g_sys.oper, sizeof(g_sys.oper), "%s", oper);
        pthread_mutex_unlock(&g_mu);
    }

    if (runCmd("timeout 8 termux-telephony-cellinfo 2>/dev/null", buf, sizeof(buf))) {
        // percorre os objetos { ... } e usa o que estiver "registered": true
        int depth = 0;
        const char* start = NULL;
        for (const char* p = buf; *p; p++) {
            if (*p == '{') { if (depth == 0) start = p; depth++; }
            else if (*p == '}' && depth > 0) {
                depth--;
                if (depth == 0 && start) {
                    size_t len = (size_t)(p - start) + 1;
                    if (len > 4000) len = 4000;
                    char obj[4100];
                    memcpy(obj, start, len);
                    obj[len] = 0;
                    if (strstr(obj, "\"registered\": true") || strstr(obj, "\"registered\":true")) {
                        int lvl = -1;
                        if (jsonNumber(obj, "level", &v)) lvl = (int)v;
                        else if (jsonNumber(obj, "dbm", &v)) {
                            lvl = v >= -85 ? 4 : (v >= -95 ? 3 : (v >= -105 ? 2 : (v >= -115 ? 1 : 0)));
                        }
                        if (lvl >= 0) {
                            if (lvl > 4) lvl = 4;
                            pthread_mutex_lock(&g_mu);
                            g_sys.mobLevel = lvl;
                            pthread_mutex_unlock(&g_mu);
                        }
                        break;
                    }
                }
            }
        }
    }

    if (tick % 3 == 0 && nowMs() - g_userTouchVol > 15000 &&
        runCmd("timeout 6 termux-volume 2>/dev/null", buf, sizeof(buf))) {
        const char* m = strstr(buf, "\"music\"");
        if (m) {
            const char* a = m;
            while (a > buf && *a != '{') a--;
            const char* b = strchr(m, '}');
            if (b && *a == '{') {
                size_t len = (size_t)(b - a) + 1;
                if (len > 400) len = 400;
                memcpy(tmp, a, len < sizeof(tmp) ? len : sizeof(tmp) - 1);
                tmp[len < sizeof(tmp) ? len : sizeof(tmp) - 1] = 0;
                double vol, mx;
                if (jsonNumber(tmp, "volume", &vol) && jsonNumber(tmp, "max_volume", &mx) && mx > 0) {
                    pthread_mutex_lock(&g_mu);
                    g_sys.volMax = (int)mx;
                    g_sys.volPct = (int)(vol * 100.0 / mx + 0.5);
                    pthread_mutex_unlock(&g_mu);
                }
            }
        }
    }
}

static void* pollThread(void*) {
    int tick = 0;
    while (true) {
        pollOnce(tick++);
        sleep(4);
    }
    return NULL;
}

static void sh(const char* cmd) {
    char b[600];
    snprintf(b, sizeof(b), "%s >/dev/null 2>&1 &", cmd);
    if (system(b) != 0) { /* ignora */ }
}

// ============================================================
//  Layout (tudo escala por "u")
// ============================================================
struct Layout {
    float W, H, u, tbY, tbH;
    Rect logoBtn, wifiBtn, mobBtn, volBtn, briBtn, clockRect, dockPanel, dockIcon;
};
static Layout L;

static void computeLayout(float W, float H) {
    L.W = W; L.H = H;
    L.u = fminf(H / 1080.0f, W / 1920.0f);
    float u = L.u;
    L.tbH = 101 * u;
    L.tbY = H - L.tbH;
    L.logoBtn   = Rect{6 * u, L.tbY + 4 * u, 100 * u, L.tbH - 8 * u};
    L.clockRect = Rect{W - 17 * u - 130 * u, L.tbY, 130 * u, L.tbH};
    float bx = L.clockRect.x - 10 * u;
    L.wifiBtn = Rect{bx - 64 * u, L.tbY + 8 * u, 64 * u, L.tbH - 16 * u};  bx = L.wifiBtn.x - 4 * u;
    L.mobBtn  = Rect{bx - 100 * u, L.tbY + 8 * u, 100 * u, L.tbH - 16 * u}; bx = L.mobBtn.x - 4 * u;
    L.volBtn  = Rect{bx - 64 * u, L.tbY + 8 * u, 64 * u, L.tbH - 16 * u};  bx = L.volBtn.x - 4 * u;
    L.briBtn  = Rect{bx - 64 * u, L.tbY + 8 * u, 64 * u, L.tbH - 16 * u};
    float dw = W * 0.4608f;
    L.dockPanel = Rect{(W - dw) * 0.5f, H - 163 * u, dw, 223 * u};
    L.dockIcon  = Rect{W * 0.5f - 46 * u, H - 147 * u, 92 * u, 92 * u};
}

// ============================================================
//  Estado da interface
// ============================================================
enum { P_NONE = 0, P_START, P_WIFI, P_MOB, P_VOL, P_BRI };
enum { D_NONE = 0, D_MOVE, D_RESIZE, D_SLIDER };
enum { E_L = 1, E_R = 2, E_T = 4, E_B = 8 };

struct FFWin { bool open, min, max; float x, y, w, h, rx, ry, rw, rh; };
static FFWin ff = {false, false, false, 0, 0, 0, 0, 0, 0, 0, 0};
static bool  g_ffPlaced = false;

static int   g_popup = P_NONE, g_drag = D_NONE, g_edges = 0, g_sliderKind = 0;
static float g_dragDX = 0, g_dragDY = 0, g_mx0 = 0, g_my0 = 0, g_sx = 0, g_sy = 0, g_sw = 0, g_sh = 0, g_lastX = 0;
static long  g_lastVolSend = 0, g_lastBriSend = 0;
static bool  g_quit = false;

static Rect popupRect(int k) {
    float u = L.u, w = 330 * u, h = 200 * u;
    Rect a = {0, 0, 0, 0};
    switch (k) {
        case P_START: w = 360 * u; h = 176 * u; return Rect{12 * u, L.tbY - 12 * u - h, w, h};
        case P_WIFI:  a = L.wifiBtn; h = 200 * u; break;
        case P_MOB:   a = L.mobBtn;  h = 200 * u; break;
        case P_VOL:   a = L.volBtn;  h = 124 * u; break;
        case P_BRI:   a = L.briBtn;  h = 124 * u; break;
        default: return Rect{0, 0, 0, 0};
    }
    float x = a.x + a.w * 0.5f - w * 0.5f;
    if (x > L.W - 8 * u - w) x = L.W - 8 * u - w;
    if (x < 8 * u) x = 8 * u;
    return Rect{x, L.tbY - 12 * u - h, w, h};
}
static Rect sliderTrack(const Rect& p) { float u = L.u; return Rect{p.x + 28 * u, p.y + p.h - 44 * u, p.w - 56 * u, 10 * u}; }
static Rect popupButton(const Rect& p) { float u = L.u; return Rect{p.x + 24 * u, p.y + p.h - 64 * u, p.w - 48 * u, 44 * u}; }
static Rect startItem(const Rect& p, int i) { float u = L.u; return Rect{p.x + 12 * u, p.y + 12 * u + i * 80 * u, p.w - 24 * u, 72 * u}; }

struct WinRects { Rect frame, title, btnMin, btnMax, btnClose, tab, toolbar, back, fwd, reload, urlbar, content; };
static WinRects winRects() {
    float u = L.u;
    WinRects R;
    R.frame    = Rect{ff.x, ff.y, ff.w, ff.h};
    R.title    = Rect{ff.x, ff.y, ff.w, 46 * u};
    R.btnClose = Rect{ff.x + ff.w - 46 * u, ff.y, 46 * u, 46 * u};
    R.btnMax   = Rect{R.btnClose.x - 46 * u, ff.y, 46 * u, 46 * u};
    R.btnMin   = Rect{R.btnMax.x - 46 * u, ff.y, 46 * u, 46 * u};
    R.tab      = Rect{ff.x + 12 * u, ff.y + 8 * u, 220 * u, 38 * u};
    R.toolbar  = Rect{ff.x, ff.y + 46 * u, ff.w, 52 * u};
    R.back     = Rect{ff.x + 12 * u,  ff.y + 52 * u, 40 * u, 40 * u};
    R.fwd      = Rect{ff.x + 58 * u,  ff.y + 52 * u, 40 * u, 40 * u};
    R.reload   = Rect{ff.x + 104 * u, ff.y + 52 * u, 40 * u, 40 * u};
    R.urlbar   = Rect{ff.x + 156 * u, ff.y + 54 * u, ff.w - 156 * u - 16 * u, 36 * u};
    R.content  = Rect{ff.x, ff.y + 98 * u, ff.w, ff.h - 98 * u};
    return R;
}

static void openFirefox() {
    if (!g_ffPlaced) {
        ff.w = L.W * 0.58f;
        ff.h = L.tbY * 0.74f;
        ff.x = (L.W - ff.w) * 0.5f;
        ff.y = L.tbY * 0.08f;
        g_ffPlaced = true;
    }
    ff.open = true;
    ff.min = false;
}

static void toggleMax() {
    if (!ff.max) { ff.rx = ff.x; ff.ry = ff.y; ff.rw = ff.w; ff.rh = ff.h; ff.max = true; }
    else { ff.max = false; ff.x = ff.rx; ff.y = ff.ry; ff.w = ff.rw; ff.h = ff.rh; }
}

// ============================================================
//  Volume / brilho
// ============================================================
static void applySlider(int kind, bool force) {
    long now = nowMs();
    char cmd[128];
    pthread_mutex_lock(&g_mu);
    int vol = g_sys.volPct, vmax = g_sys.volMax, bri = g_sys.briPct;
    pthread_mutex_unlock(&g_mu);
    if (kind == P_VOL) {
        if (!force && now - g_lastVolSend < 200) return;
        g_lastVolSend = now;
        snprintf(cmd, sizeof(cmd), "termux-volume music %d", (int)(vol * vmax / 100.0f + 0.5f));
        sh(cmd);
    } else if (kind == P_BRI) {
        if (!force && now - g_lastBriSend < 200) return;
        g_lastBriSend = now;
        int v = (int)(bri * 255 / 100.0f + 0.5f);
        if (v < 1) v = 1;
        snprintf(cmd, sizeof(cmd), "termux-brightness %d", v);
        sh(cmd);
    }
}

static void setSliderFromX(float x, bool final) {
    Rect t = sliderTrack(popupRect(g_sliderKind));
    float pct = (x - t.x) / t.w * 100.0f;
    if (pct < 0) pct = 0;
    if (pct > 100) pct = 100;
    pthread_mutex_lock(&g_mu);
    if (g_sliderKind == P_VOL) { g_sys.volPct = (int)(pct + 0.5f); g_userTouchVol = nowMs(); }
    else g_sys.briPct = (int)(pct + 0.5f);
    pthread_mutex_unlock(&g_mu);
    applySlider(g_sliderKind, final);
}

// ============================================================
//  Entrada (mouse / toque)
// ============================================================
static void popupPress(float x, float y) {
    Rect p = popupRect(g_popup);
    if (g_popup == P_START) {
        if (inRect(startItem(p, 0), x, y)) { openFirefox(); g_popup = P_NONE; }
        else if (inRect(startItem(p, 1), x, y)) g_quit = true;
    } else if (g_popup == P_WIFI) {
        if (inRect(popupButton(p), x, y)) { sh("am start -a android.settings.WIFI_SETTINGS"); g_popup = P_NONE; }
    } else if (g_popup == P_MOB) {
        if (inRect(popupButton(p), x, y)) { sh("am start -a android.settings.DATA_ROAMING_SETTINGS"); g_popup = P_NONE; }
    } else if (g_popup == P_VOL || g_popup == P_BRI) {
        Rect t = sliderTrack(p);
        Rect hit = Rect{t.x - 20 * L.u, t.y - 26 * L.u, t.w + 40 * L.u, t.h + 52 * L.u};
        if (inRect(hit, x, y)) { g_drag = D_SLIDER; g_sliderKind = g_popup; setSliderFromX(x, false); }
    }
}

static void windowPress(float x, float y) {
    if (!ff.open || ff.min) return;
    float u = L.u;
    WinRects R = winRects();
    float m = 10 * u;
    bool inExt = x >= ff.x - m && x <= ff.x + ff.w + m && y >= ff.y - m && y <= ff.y + ff.h + m;
    if (!inExt) return;

    if (inRect(R.btnClose, x, y)) { ff.open = false; if (ff.max) toggleMax(); return; }
    if (inRect(R.btnMax, x, y))   { toggleMax(); return; }
    if (inRect(R.btnMin, x, y))   { ff.min = true; return; }

    if (!ff.max) {
        int e = 0;
        if (x < ff.x + m) e |= E_L;
        if (x > ff.x + ff.w - m) e |= E_R;
        if (y < ff.y + m) e |= E_T;
        if (y > ff.y + ff.h - m) e |= E_B;
        if (e) {
            g_drag = D_RESIZE; g_edges = e;
            g_mx0 = x; g_my0 = y; g_sx = ff.x; g_sy = ff.y; g_sw = ff.w; g_sh = ff.h;
            return;
        }
    }
    if (inRect(R.title, x, y) && !ff.max) {
        g_drag = D_MOVE; g_dragDX = x - ff.x; g_dragDY = y - ff.y;
    }
}

static void mouseDown(float x, float y) {
    int prev = g_popup;
    if (g_popup) {
        if (inRect(popupRect(g_popup), x, y)) { popupPress(x, y); return; }
        g_popup = P_NONE;
    }
    // dock (área cinza do meio)
    if (inRect(L.dockIcon, x, y)) {
        if (!ff.open || ff.min) openFirefox();
        else ff.min = true;
        return;
    }
    if (inRect(L.dockPanel, x, y)) return;

    // barra de tarefas
    if (y >= L.tbY) {
        if (inRect(L.logoBtn, x, y))      { if (prev != P_START) g_popup = P_START; }
        else if (inRect(L.briBtn, x, y))  { if (prev != P_BRI)   g_popup = P_BRI; }
        else if (inRect(L.volBtn, x, y))  { if (prev != P_VOL)   g_popup = P_VOL; }
        else if (inRect(L.mobBtn, x, y))  { if (prev != P_MOB)   g_popup = P_MOB; }
        else if (inRect(L.wifiBtn, x, y)) { if (prev != P_WIFI)  g_popup = P_WIFI; }
        return;
    }
    windowPress(x, y);
}

static void doResize(float x, float y) {
    float u = L.u, minW = 420 * u, minH = 300 * u;
    float dx = x - g_mx0, dy = y - g_my0;
    float nx = g_sx, ny = g_sy, nw = g_sw, nh = g_sh;
    if (g_edges & E_R) nw = g_sw + dx;
    if (g_edges & E_B) nh = g_sh + dy;
    if (g_edges & E_L) nw = g_sw - dx;
    if (g_edges & E_T) nh = g_sh - dy;
    if (nw < minW) nw = minW;
    if (nh < minH) nh = minH;
    if (g_edges & E_L) nx = g_sx + g_sw - nw;
    if (g_edges & E_T) ny = g_sy + g_sh - nh;
    if ((g_edges & E_T) && ny < 0) { nh += ny; ny = 0; }
    ff.x = nx; ff.y = ny; ff.w = nw; ff.h = nh;
}

static void mouseMove(float x, float y) {
    g_lastX = x;
    if (g_drag == D_SLIDER) setSliderFromX(x, false);
    else if (g_drag == D_RESIZE) doResize(x, y);
    else if (g_drag == D_MOVE) {
        float u = L.u;
        float nx = x - g_dragDX, ny = y - g_dragDY;
        if (ny < 0) ny = 0;
        if (ny > L.tbY - 46 * u) ny = L.tbY - 46 * u;
        if (nx < -(ff.w - 120 * u)) nx = -(ff.w - 120 * u);
        if (nx > L.W - 120 * u) nx = L.W - 120 * u;
        ff.x = nx; ff.y = ny;
    }
}

static void mouseUp(float x, float y) {
    (void)y;
    if (g_drag == D_SLIDER) setSliderFromX(x, true);
    g_drag = D_NONE;
}

// ============================================================
//  Desenho: ícones da barra
// ============================================================
static void drawFoxVector(float cx, float cy, float r) {
    circle(cx, cy, r, C(0.98f, 0.52f, 0.08f));
    circle(cx + r * 0.14f, cy - r * 0.04f, r * 0.80f, C(0.36f, 0.20f, 0.64f));
    circle(cx + r * 0.32f, cy - r * 0.22f, r * 0.34f, C(0.55f, 0.35f, 0.85f));
    circle(cx - r * 0.55f, cy - r * 0.55f, r * 0.20f, C(1.00f, 0.80f, 0.20f));
}

static void drawFox(float x, float y, float size, Color tint) {
    if (g_iconFox.ok) drawIcon(g_iconFox, x, y, size, size, tint);
    else drawFoxVector(x + size * 0.5f, y + size * 0.5f, size * 0.46f);
}

static void drawAndroid(float x, float y, float k, Color body, Color eye) {
    ringSector(x + 30 * k, y + 24 * k, 0, 22 * k, PI_F, 2.0f * PI_F, 24, body);
    capsule(x + 21 * k, y + 7 * k, x + 15 * k, y, 1.6f * k, body);
    capsule(x + 39 * k, y + 7 * k, x + 45 * k, y, 1.6f * k, body);
    circle(x + 22 * k, y + 15 * k, 2.6f * k, eye);
    circle(x + 38 * k, y + 15 * k, 2.6f * k, eye);
    rrect(x + 8 * k, y + 27 * k, 44 * k, 30 * k, 5 * k, body);
    capsule(x + 3.5f * k, y + 31 * k, x + 3.5f * k, y + 48 * k, 3.5f * k, body);
    capsule(x + 56.5f * k, y + 31 * k, x + 56.5f * k, y + 48 * k, 3.5f * k, body);
    rrect(x + 17 * k, y + 50 * k, 9 * k, 14 * k, 4 * k, body);
    rrect(x + 34 * k, y + 50 * k, 9 * k, 14 * k, 4 * k, body);
}

static void drawWifi(float cx, float cy, float u, int level) {
    float ox = cx, oy = cy + 14 * u;
    circle(ox, oy, 3.2f * u, level >= 0 ? WHITE : DIM);
    for (int i = 0; i < 3; i++) {
        float rc = (11.0f + 8.5f * i) * u, th = 3.6f * u;
        Color c = (level >= i + 1) ? WHITE : DIM;
        ringSector(ox, oy, rc - th * 0.5f, rc + th * 0.5f, -2.25f, -0.89f, 14, c);
    }
    if (level < 0) capsule(cx - 15 * u, cy - 14 * u, cx + 15 * u, cy + 16 * u, 1.6f * u, WHITE);
}

static void drawBars(float x, float cy, float u, int level) {
    float bw = 6 * u, gap = 3.5f * u, bottom = cy + 14 * u;
    for (int i = 0; i < 4; i++) {
        float h = (9 + 6 * i) * u;
        rrect(x + i * (bw + gap), bottom - h, bw, h, 1.5f * u, (level >= i + 1) ? WHITE : DIM);
    }
}

static void drawVolume(float cx, float cy, float u, int pct) {
    float x = cx - 17 * u;
    rrect(x, cy - 6 * u, 8 * u, 12 * u, 1.5f * u, WHITE);
    tri(x + 7 * u, cy - 6 * u, x + 7 * u, cy + 6 * u, x + 19 * u, cy + 13 * u, WHITE);
    tri(x + 7 * u, cy - 6 * u, x + 19 * u, cy + 13 * u, x + 19 * u, cy - 13 * u, WHITE);
    if (pct <= 0) {
        capsule(x + 24 * u, cy - 6 * u, x + 34 * u, cy + 6 * u, 1.5f * u, WHITE);
        capsule(x + 24 * u, cy + 6 * u, x + 34 * u, cy - 6 * u, 1.5f * u, WHITE);
    } else {
        ringSector(x + 19 * u, cy, 8 * u, 11 * u, -0.7f, 0.7f, 8, WHITE);
        ringSector(x + 19 * u, cy, 15 * u, 18 * u, -0.7f, 0.7f, 10, pct > 50 ? WHITE : DIM);
    }
}

static void drawSun(float cx, float cy, float u) {
    circle(cx, cy, 6.2f * u, WHITE);
    for (int i = 0; i < 8; i++) {
        float a = i * PI_F / 4.0f;
        capsule(cx + 10 * u * cosf(a), cy + 10 * u * sinf(a), cx + 15 * u * cosf(a), cy + 15 * u * sinf(a), 1.5f * u, WHITE);
    }
}

static void drawSlider(const Rect& t, int pct) {
    float u = L.u;
    rrect(t.x, t.y, t.w, t.h, t.h * 0.5f, C(0.34f, 0.34f, 0.40f));
    float fw = t.w * pct / 100.0f;
    rrect(t.x, t.y, fw < t.h ? t.h : fw, t.h, t.h * 0.5f, C(0.30f, 0.62f, 1.0f));
    circle(t.x + fw, t.y + t.h * 0.5f, 13 * u, WHITE);
}

static void drawButton(const Rect& r, const char* label) {
    float u = L.u;
    rrect(r.x, r.y, r.w, r.h, 10 * u, ACCENT);
    drawTextCenter(label, r.x + r.w * 0.5f, r.y + r.h * 0.5f, 21 * u, true, WHITE);
}

// ============================================================
//  Desenho: janela do Firefox
// ============================================================
static void drawChevron(float cx, float cy, float u, int dir) {
    capsule(cx + dir * 3 * u, cy - 7 * u, cx - dir * 4 * u, cy, 1.6f * u, WHITE);
    capsule(cx - dir * 4 * u, cy, cx + dir * 3 * u, cy + 7 * u, 1.6f * u, WHITE);
}

static void drawFirefoxWindow() {
    float u = L.u;
    WinRects R = winRects();
    float rad = ff.max ? 0.0f : 14 * u;
    Color strip = C(0.09f, 0.085f, 0.11f), bar = C(0.17f, 0.165f, 0.20f), page = C(0.11f, 0.105f, 0.13f);

    if (!ff.max) rrect(ff.x - 6 * u, ff.y + 2 * u, ff.w + 12 * u, ff.h + 14 * u, rad + 6 * u, C(0, 0, 0, 0.20f));
    rrect(ff.x, ff.y, ff.w, ff.h, rad, strip);

    // aba ativa
    rrect(R.tab.x, R.tab.y, R.tab.w, R.tab.h + 12 * u, 10 * u, bar);
    rect(R.toolbar.x, R.toolbar.y, R.toolbar.w, R.toolbar.h, bar);
    drawFox(R.tab.x + 10 * u, R.tab.y + 8 * u, 22 * u, WHITE);
    drawText("Nova aba", R.tab.x + 42 * u, R.tab.y + 19 * u, 17 * u, false, WHITE);
    drawText("\xC3\x97", R.tab.x + R.tab.w - 26 * u, R.tab.y + 19 * u, 22 * u, false, GRAYTXT);

    // botões da janela
    float by = ff.y + 23 * u;
    rect(R.btnMin.x + 17 * u, by + 5 * u, 12 * u, 1.8f * u, WHITE);
    float mx = R.btnMax.x + 17 * u, my = by - 6 * u;
    rect(mx, my, 12 * u, 1.8f * u, WHITE);  rect(mx, my + 10.2f * u, 12 * u, 1.8f * u, WHITE);
    rect(mx, my, 1.8f * u, 12 * u, WHITE);  rect(mx + 10.2f * u, my, 1.8f * u, 12 * u, WHITE);
    float cx = R.btnClose.x + 23 * u;
    capsule(cx - 6 * u, by - 6 * u, cx + 6 * u, by + 6 * u, 1.2f * u, WHITE);
    capsule(cx - 6 * u, by + 6 * u, cx + 6 * u, by - 6 * u, 1.2f * u, WHITE);

    // barra de ferramentas
    drawChevron(R.back.x + 20 * u, R.back.y + 20 * u, u, 1);
    drawChevron(R.fwd.x + 20 * u, R.fwd.y + 20 * u, u, -1);
    ringSector(R.reload.x + 20 * u, R.reload.y + 20 * u, 6 * u, 8.2f * u, -1.2f, 3.6f, 20, WHITE);
    tri(R.reload.x + 20 * u + 8 * u, R.reload.y + 20 * u - 8 * u, R.reload.x + 20 * u + 13 * u, R.reload.y + 20 * u - 1 * u,
        R.reload.x + 20 * u + 3 * u, R.reload.y + 20 * u - 2 * u, WHITE);
    rrect(R.urlbar.x, R.urlbar.y, R.urlbar.w, R.urlbar.h, 18 * u, page);
    drawText("Pesquisar ou digitar endereço", R.urlbar.x + 18 * u, R.urlbar.y + R.urlbar.h * 0.5f, 18 * u, false, GRAYTXT);

    // conteúdo
    rrect(R.content.x, R.content.y, R.content.w, R.content.h, rad, page);
    rect(R.content.x, R.content.y, R.content.w, rad + 2 * u, page);
    float ccx = R.content.x + R.content.w * 0.5f, ccy = R.content.y + R.content.h * 0.5f;
    float isz = fminf(130 * u, R.content.h * 0.38f);
    drawFox(ccx - isz * 0.5f, ccy - isz * 0.5f - 22 * u, isz, WHITE);
    drawTextCenter("Mozilla Firefox", ccx, ccy + isz * 0.5f + 12 * u, 32 * u, true, WHITE);
    drawTextCenter("Janela do DroidOS", ccx, ccy + isz * 0.5f + 48 * u, 19 * u, false, GRAYTXT);

    // alça de redimensionar
    if (!ff.max) {
        float gx = ff.x + ff.w - 8 * u, gy = ff.y + ff.h - 8 * u;
        for (int i = 1; i <= 3; i++)
            capsule(gx - i * 5 * u, gy, gx, gy - i * 5 * u, 0.9f * u, C(1, 1, 1, 0.30f));
    }
}

// ============================================================
//  Desenho: popups
// ============================================================
static void drawPopup(int k, const SysInfo& S) {
    float u = L.u;
    Rect p = popupRect(k);
    rrect(p.x - 4 * u, p.y + 2 * u, p.w + 8 * u, p.h + 10 * u, 22 * u, C(0, 0, 0, 0.22f));
    rrect(p.x, p.y, p.w, p.h, 18 * u, POPUP_BG);
    char line[160];

    if (k == P_START) {
        Rect a = startItem(p, 0), b = startItem(p, 1);
        rrect(a.x, a.y, a.w, a.h, 12 * u, ROW_BG);
        drawFox(a.x + 14 * u, a.y + 12 * u, 48 * u, WHITE);
        drawText("Firefox", a.x + 78 * u, a.y + a.h * 0.5f, 26 * u, true, WHITE);
        rrect(b.x, b.y, b.w, b.h, 12 * u, ROW_BG);
        circle(b.x + 38 * u, b.y + b.h * 0.5f, 18 * u, C(0.85f, 0.25f, 0.25f));
        ringSector(b.x + 38 * u, b.y + b.h * 0.5f + 1 * u, 7 * u, 9 * u, -0.97f, 4.11f, 20, WHITE);
        rect(b.x + 37 * u, b.y + b.h * 0.5f - 10 * u, 2 * u, 10 * u, WHITE);
        drawText("Encerrar DroidOS", b.x + 78 * u, b.y + b.h * 0.5f, 24 * u, false, WHITE);
    } else if (k == P_WIFI) {
        drawText("Wi-Fi", p.x + 24 * u, p.y + 34 * u, 27 * u, true, WHITE);
        if (S.wifiLevel < 0) snprintf(line, sizeof(line), "Não conectado");
        else snprintf(line, sizeof(line), "%s", S.ssid[0] ? S.ssid : "Rede conectada");
        drawText(line, p.x + 24 * u, p.y + 78 * u, 22 * u, false, WHITE);
        const char* q = S.wifiLevel == 3 ? "Forte" : S.wifiLevel == 2 ? "Média" : S.wifiLevel == 1 ? "Fraca" : S.wifiLevel == 0 ? "Muito fraca" : "Sem sinal";
        if (S.wifiLevel >= 0) snprintf(line, sizeof(line), "Sinal: %d dBm (%s)", S.rssi, q);
        else snprintf(line, sizeof(line), "Sinal: %s", q);
        drawText(line, p.x + 24 * u, p.y + 110 * u, 18 * u, false, GRAYTXT);
        drawButton(popupButton(p), "Configurações de Wi-Fi");
    } else if (k == P_MOB) {
        drawText("Dados móveis", p.x + 24 * u, p.y + 34 * u, 27 * u, true, WHITE);
        snprintf(line, sizeof(line), "Rede: %s", S.netGen);
        drawText(line, p.x + 24 * u, p.y + 78 * u, 22 * u, false, WHITE);
        snprintf(line, sizeof(line), "%s  \xC2\xB7  Sinal %d/4", S.oper[0] ? S.oper : "Operadora", S.mobLevel);
        drawText(line, p.x + 24 * u, p.y + 110 * u, 18 * u, false, GRAYTXT);
        drawButton(popupButton(p), "Configurações de rede");
    } else if (k == P_VOL || k == P_BRI) {
        int pct = (k == P_VOL) ? S.volPct : S.briPct;
        drawText(k == P_VOL ? "Volume" : "Brilho", p.x + 24 * u, p.y + 34 * u, 27 * u, true, WHITE);
        snprintf(line, sizeof(line), "%d%%", pct);
        drawTextRight(line, p.x + p.w - 24 * u, p.y + 34 * u, 24 * u, false, WHITE);
        drawSlider(sliderTrack(p), pct);
    }
}

// ============================================================
//  Cena completa (ordem = ordem de desenho)
// ============================================================
static void buildScene(float W, float H) {
    g_W = W; g_H = H; g_count = 0;
    computeLayout(W, H);
    float u = L.u;

    SysInfo S;
    pthread_mutex_lock(&g_mu);
    S = g_sys;
    pthread_mutex_unlock(&g_mu);

    if (ff.max) { ff.x = 0; ff.y = 0; ff.w = W; ff.h = L.tbY; }

    // 1. janela do Firefox (fica atrás da barra e do dock)
    if (ff.open && !ff.min) drawFirefoxWindow();

    // 2. barra de tarefas
    rect(0, L.tbY, W, L.tbH, TASKBAR);

    // logo do personagem (esquerda)
    float lsz = 82 * u;
    float lx = 14 * u, ly = L.tbY + (L.tbH - lsz) * 0.5f;
    if (g_iconLogo.ok) drawIcon(g_iconLogo, lx, ly, lsz, lsz, WHITE);
    else drawAndroid(18 * u, L.tbY + (L.tbH - 64 * u) * 0.5f, u, WHITE, TASKBAR);

    // status: brilho, volume, dados móveis, Wi-Fi
    float cy = L.tbY + L.tbH * 0.5f;
    drawSun(L.briBtn.x + L.briBtn.w * 0.5f, cy, u);
    drawVolume(L.volBtn.x + L.volBtn.w * 0.5f, cy, u, S.volPct);
    drawBars(L.mobBtn.x + 8 * u, cy, u, S.mobLevel);
    drawText(S.netGen, L.mobBtn.x + 8 * u + 4 * (6 + 3.5f) * u + 4 * u, cy, 21 * u, true, WHITE);
    drawWifi(L.wifiBtn.x + L.wifiBtn.w * 0.5f, cy, u, S.wifiLevel);

    // relógio e data
    time_t now = time(NULL);
    struct tm* tmv = localtime(&now);
    char hora[16], data[16];
    strftime(hora, sizeof(hora), "%H:%M", tmv);
    strftime(data, sizeof(data), "%d/%m/%Y", tmv);
    float right = W - 17 * u;
    drawTextRight(hora, right, L.tbY + 33 * u, 44 * u, true, WHITE);
    drawTextRight(data, right, L.tbY + 77 * u, 26 * u, true, WHITE);

    // 3. dock (área cinza do meio, por cima da barra) com o Firefox
    rrect(L.dockPanel.x, L.dockPanel.y, L.dockPanel.w, L.dockPanel.h, 22 * u, WINDOW_BG);
    drawFox(L.dockIcon.x, L.dockIcon.y, L.dockIcon.w, WHITE);
    drawTextCenter("Firefox", W * 0.5f, L.dockIcon.y + L.dockIcon.h + 22 * u, 18 * u, false, WHITE);
    if (ff.open) rrect(W * 0.5f - 14 * u, H - 13 * u, 28 * u, 4 * u, 2 * u, ff.min ? GRAYTXT : WHITE);

    // 4. popup aberto
    if (g_popup) drawPopup(g_popup, S);
}

// ============================================================
//  main
// ============================================================
int main() {
    Display* xDisplay = XOpenDisplay(NULL);
    if (!xDisplay) return -1;

    int screen = DefaultScreen(xDisplay);
    Window rootWin = RootWindow(xDisplay, screen);

    XWindowAttributes gwa;
    XGetWindowAttributes(xDisplay, rootWin, &gwa);
    int screenWidth = gwa.width;
    int screenHeight = gwa.height;

    Window xWindow = XCreateSimpleWindow(
        xDisplay, rootWin,
        0, 0, screenWidth, screenHeight, 0,
        BlackPixel(xDisplay, screen),
        WhitePixel(xDisplay, screen)
    );

    XStoreName(xDisplay, xWindow, "DroidOS");
    XSelectInput(xDisplay, xWindow, ButtonPressMask | ButtonReleaseMask | ButtonMotionMask | KeyPressMask | ExposureMask);
    XMapWindow(xDisplay, xWindow);
    XFlush(xDisplay);

    EGLDisplay display = eglGetDisplay((EGLNativeDisplayType)xDisplay);
    eglInitialize(display, NULL, NULL);

    EGLConfig config;
    EGLint numConfigs;
    EGLint attribs[] = {
        EGL_RENDERABLE_TYPE, EGL_OPENGL_ES3_BIT,
        EGL_SURFACE_TYPE, EGL_WINDOW_BIT,
        EGL_BLUE_SIZE, 8, EGL_GREEN_SIZE, 8, EGL_RED_SIZE, 8,
        EGL_NONE
    };

    eglChooseConfig(display, attribs, &config, 1, &numConfigs);
    EGLint contextAttribs[] = { EGL_CONTEXT_CLIENT_VERSION, 3, EGL_NONE };

    EGLContext context = eglCreateContext(display, config, EGL_NO_CONTEXT, contextAttribs);
    EGLSurface surface = eglCreateWindowSurface(display, config, (EGLNativeWindowType)xWindow, NULL);
    eglMakeCurrent(display, surface, surface, context);

    GLuint vertexShader = glCreateShader(GL_VERTEX_SHADER);
    glShaderSource(vertexShader, 1, &vertexShaderSource, NULL);
    glCompileShader(vertexShader);

    GLuint fragmentShader = glCreateShader(GL_FRAGMENT_SHADER);
    glShaderSource(fragmentShader, 1, &fragmentShaderSource, NULL);
    glCompileShader(fragmentShader);

    GLuint shaderProgram = glCreateProgram();
    glAttachShader(shaderProgram, vertexShader);
    glAttachShader(shaderProgram, fragmentShader);
    glLinkProgram(shaderProgram);

    GLint scaleLoc = glGetUniformLocation(shaderProgram, "uScale");
    GLint offsetLoc = glGetUniformLocation(shaderProgram, "uOffset");
    GLint texLoc = glGetUniformLocation(shaderProgram, "uTex");

    // ---- atlas: pixel branco, fontes e ícones ----
    g_atlas = (unsigned char*)calloc((size_t)ATLAS_W * ATLAS_H * 4, 1);
    for (int y = 0; y < 8; y++)
        for (int x = 0; x < 8; x++)
            memset(g_atlas + ((size_t)y * ATLAS_W + x) * 4, 255, 4);

    initFonts();

    char path[600];
    if (findAsset("logo.png", path, sizeof(path))) loadIconTo(path, 0, 1056, 256, &g_iconLogo);
    if (findAsset("firefox.png", path, sizeof(path))) loadIconTo(path, 256, 1056, 256, &g_iconFox);

    GLuint tex;
    glGenTextures(1, &tex);
    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, tex);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, ATLAS_W, ATLAS_H, 0, GL_RGBA, GL_UNSIGNED_BYTE, g_atlas);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);

    GLuint VAO, VBO;
    glGenVertexArrays(1, &VAO);
    glGenBuffers(1, &VBO);

    glBindVertexArray(VAO);
    glBindBuffer(GL_ARRAY_BUFFER, VBO);
    glBufferData(GL_ARRAY_BUFFER, sizeof(float) * 48, NULL, GL_DYNAMIC_DRAW);

    glVertexAttribPointer(0, 2, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)0);
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(1, 4, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)(2 * sizeof(float)));
    glEnableVertexAttribArray(1);
    glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 8 * sizeof(float), (void*)(6 * sizeof(float)));
    glEnableVertexAttribArray(2);

    glEnable(GL_BLEND);
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);

    computeLayout((float)screenWidth, (float)screenHeight);

    pthread_t th;
    if (pthread_create(&th, NULL, pollThread, NULL) == 0) pthread_detach(th);

    while (!g_quit) {
        // ---- eventos ----
        while (XPending(xDisplay)) {
            XEvent ev;
            XNextEvent(xDisplay, &ev);
            switch (ev.type) {
                case ButtonPress:
                    if (ev.xbutton.button == 1) mouseDown((float)ev.xbutton.x, (float)ev.xbutton.y);
                    break;
                case ButtonRelease:
                    if (ev.xbutton.button == 1) mouseUp((float)ev.xbutton.x, (float)ev.xbutton.y);
                    break;
                case MotionNotify:
                    mouseMove((float)ev.xmotion.x, (float)ev.xmotion.y);
                    break;
                case KeyPress:
                    if (XLookupKeysym(&ev.xkey, 0) == XK_Escape) g_popup = P_NONE;
                    break;
                default:
                    break;
            }
        }

        // ---- desenho ----
        glViewport(0, 0, screenWidth, screenHeight);
        glClearColor(WALLPAPER.r, WALLPAPER.g, WALLPAPER.b, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT);

        glUseProgram(shaderProgram);
        glUniform2f(scaleLoc, 1.0f, 1.0f);
        glUniform2f(offsetLoc, 0.0f, 0.0f);
        glUniform1i(texLoc, 0);

        glBindVertexArray(VAO);

        buildScene((float)screenWidth, (float)screenHeight);
        int nVerts = (g_count / 8) / 3 * 3;

        glBindBuffer(GL_ARRAY_BUFFER, VBO);
        glBufferData(GL_ARRAY_BUFFER, sizeof(float) * g_count, g_buf, GL_DYNAMIC_DRAW);
        glDrawArrays(GL_TRIANGLES, 0, nVerts);

        eglSwapBuffers(display, surface);
        usleep(16000);
    }

    eglMakeCurrent(display, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT);
    eglDestroySurface(display, surface);
    eglDestroyContext(display, context);
    eglTerminate(display);
    XDestroyWindow(xDisplay, xWindow);
    XCloseDisplay(xDisplay);
    return 0;
}
DROIDOS_MAIN_EOF

# ---- script para abrir depois ----
cat > run.sh << 'DROIDOS_RUN_EOF'
#!/usr/bin/env bash
# Compila (se o código mudou) e executa o DroidOS.
cd "$(dirname "$0")" || exit 1
export DROIDOS_DIR="$PWD"
export DISPLAY="${DISPLAY:-:0}"

# variáveis extras (GPU etc.): coloque em env.sh, por exemplo:  export GALLIUM_DRIVER=zink
if [ -f env.sh ]; then
  . ./env.sh
elif [ -f "$HOME/test_local/run_engine.sh" ]; then
  # reaproveita só os "export" do seu run_engine.sh antigo
  eval "$(grep -E '^[[:space:]]*export[[:space:]]' "$HOME/test_local/run_engine.sh")"
fi

[ "$1" = "--rebuild" ] && rm -f build/droidos

# DROIDOS_START_X11=1 ~/droidos/run.sh  -> também inicia o servidor termux-x11
if [ "$DROIDOS_START_X11" = "1" ] && command -v termux-x11 >/dev/null 2>&1; then
  termux-x11 "$DISPLAY" >/dev/null 2>&1 &
  sleep 2
fi

mkdir -p build
if [[ ! -f build/droidos || src/main.cpp -nt build/droidos ]]; then
  echo "==> Compilando..."
  g++ -O2 -w -I third_party src/main.cpp -o build/droidos -lX11 -lEGL -lGLESv2 || { echo "Erro na compilação."; exit 1; }
fi

exec ./build/droidos
DROIDOS_RUN_EOF
chmod +x run.sh

./run.sh
