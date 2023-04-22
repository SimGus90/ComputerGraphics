#include <windows.h>
#include <gl/gl.h>
#include "stb_easy_font.h"

#include <string.h>

#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"

#include <stdbool.h>

LRESULT CALLBACK WindowProc(HWND, UINT, WPARAM, LPARAM);
void EnableOpenGL(HWND hwnd, HDC*, HGLRC*);
void DisableOpenGL(HWND, HDC, HGLRC);

int WindowHeight = 600;
int WindowWeight = 800;
int State = 0; //состояние программы

//------------------------------------------------BUTTON-----------------------------------------
typedef struct {
    char name[20];
    float vert[8];
    bool hover;
    bool visible;
} Button;

Button btn[4];
int btnCnt = sizeof(btn)/sizeof(btn[0]);

BOOL pointInButton(int x, int y, Button btn)
{
    return (x > btn.vert[0]) && (x < btn.vert[4]) &&
           (y > btn.vert[1]) && (y < btn.vert[5]);
}

void buttonRender(Button *btn, char *name, int x, int y, int width, int height, float scale)
{
    strcpy(btn->name, name);

    if (btn->visible){
        float buffer[1000];
        int num_quads;

        btn->vert[0]=btn->vert[6]=x;
        btn->vert[2]=btn->vert[4]=x + width;
        btn->vert[1]=btn->vert[3]=y;
        btn->vert[5]=btn->vert[7]=y + height;

        num_quads = stb_easy_font_print(0, 0, name, 0, buffer, sizeof(buffer)); // запись координат вершин элементов имени
        float textPosX = x +(width-stb_easy_font_width(name)*scale)/2.0;
        float textPosY = y +(height-stb_easy_font_height(name)*scale)/2.0;
        textPosY+= scale*2;

        glEnableClientState(GL_VERTEX_ARRAY);
        if (btn->hover) glColor3f(1,1,1);
        else glColor3f(0.5,0.5,0.5);
        glVertexPointer(2,GL_FLOAT,0,btn->vert);
        glDrawArrays(GL_TRIANGLE_FAN,0,4);

        glColor3f(1.0,1.0,1.0); //цвет обводки
        glLineWidth(4); // толщина обводки кнопки
        glDrawArrays(GL_LINE_LOOP,0,4); //отрисовка обводки
        glDisableClientState(GL_VERTEX_ARRAY);

        glPushMatrix(); //матрицу в стек
            glEnableClientState(GL_VERTEX_ARRAY); // разрешение
            glColor3f(0.8,0.8,0.8); //цвет текста
            glTranslatef(textPosX,textPosY,0); //перенос матрицы для отрисовки текста
            glScalef(scale,scale,1); //масштабирование текста
            glVertexPointer(2, GL_FLOAT, 16, buffer); //вектор для отрисовки
            glDrawArrays(GL_QUADS, 0, num_quads*4); //отрисовка текста
            glDisableClientState(GL_VERTEX_ARRAY);
        glPopMatrix();
    }
};

//------------------------------------------------Texture-----------------------------------------
typedef struct {
    unsigned int texture;
} Texture;

Texture tex_Character;
Texture tex_Background;
Texture tex_Blocks;

void createTexture(Texture *obj, const char *path)
{
    int h_image, w_image, cnt;
    unsigned char *data = stbi_load(path, &w_image, &h_image, &cnt, 0);

    glGenTextures(1, &obj->texture);
    glBindTexture(GL_TEXTURE_2D, obj->texture);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, w_image, h_image,
            0, cnt == 4 ? GL_RGBA : GL_RGB, GL_UNSIGNED_BYTE, data);

    glBindTexture(GL_TEXTURE_2D, 0);
    stbi_image_free(data);
}

void renderAreaTexture(Texture *obj, int x, int y, float weight, float height, float areaWeight, float areaHeight, int n, int t, BOOL visible)
{
    if (visible)
    {
        static float vertex[8]; //вектор текстурируемого многоугольника
        static float TexCord[]= {0,0, 1,0, 1,1, 0,1}; // текстурные координаты изображения

        vertex[0]=vertex[6]=x;
        vertex[2]=vertex[4]=x + areaWeight;
        vertex[1]=vertex[3]=y;
        vertex[5]=vertex[7]=y + areaHeight;

        float vrtcoord_left = (areaWeight*n)/weight; //вычисление координат кадра на изображении от
        float vrtcoord_right = vrtcoord_left+(areaWeight/weight); //номера кадра
        float vrtcoord_top = (areaHeight*t)/height;
        float vrtcoord_bottom = vrtcoord_top+(areaHeight/height);

        TexCord[1] = TexCord[3] = vrtcoord_top; // запись в вектор текстурных координат
        TexCord[5] = TexCord[7] = vrtcoord_bottom;
        TexCord[2] = TexCord[4] = vrtcoord_right;
        TexCord[0] = TexCord[6] = vrtcoord_left;

        glEnable(GL_TEXTURE_2D); //разрешение использования текстуры
        glBindTexture(GL_TEXTURE_2D, obj->texture);
        glEnable(GL_ALPHA_TEST); // проверка на элементы α-канала (не обязательно)
        glAlphaFunc(GL_GREATER, 0.99); // задается типе уровня и его числовая граница

        glPushMatrix();
            glEnableClientState(GL_VERTEX_ARRAY);
            glEnableClientState(GL_TEXTURE_COORD_ARRAY);

            glColor3f(1,1,1);
            glVertexPointer(2,GL_FLOAT,0,vertex);
            glTexCoordPointer(2, GL_FLOAT, 0, TexCord);
            glDrawArrays(GL_TRIANGLE_FAN,0,4);

            glDisableClientState(GL_VERTEX_ARRAY);
            glDisableClientState(GL_TEXTURE_COORD_ARRAY);
            glDisable(GL_ALPHA_TEST);
            glDisable(GL_TEXTURE_2D);
        glPopMatrix();
    }
}

void renderFullTexture(Texture *obj, int x, int y, int weight, int height, BOOL visible)
{
    if (visible)
    {
        static float vertex[8]; //вектор текстурируемого многоугольника
        static float TexCord[]= {0,0, 1,0, 1,1, 0,1}; // текстурные координаты изображения

        vertex[0]=vertex[6]=x;
        vertex[2]=vertex[4]=x + weight;
        vertex[1]=vertex[3]=y;
        vertex[5]=vertex[7]=y + height;

        glEnable(GL_TEXTURE_2D); //разрешение использования текстуры
        glBindTexture(GL_TEXTURE_2D, obj->texture);
        glEnable(GL_ALPHA_TEST); // проверка на элементы α-канала (не обязательно)
        glAlphaFunc(GL_GREATER, 0.99); // задается типе уровня и его числовая граница

        glPushMatrix();
            glEnableClientState(GL_VERTEX_ARRAY);
            glEnableClientState(GL_TEXTURE_COORD_ARRAY);

            glColor3f(1,1,1);
            glVertexPointer(2,GL_FLOAT,0,vertex);
            glTexCoordPointer(2, GL_FLOAT, 0, TexCord);
            glDrawArrays(GL_TRIANGLE_FAN,0,4);

            glDisableClientState(GL_VERTEX_ARRAY);
            glDisableClientState(GL_TEXTURE_COORD_ARRAY);
            glDisable(GL_ALPHA_TEST);
            glDisable(GL_TEXTURE_2D);
        glPopMatrix();
    }
}

//------------------------------------------------MAIN CHARACTER-----------------------------------------
typedef struct
{
    bool onGround;
    float currentFrame;
    int currentTypeAnimation;
    float x, y, dx, dy; //обе координаты и скорость изменения
} Hero;

Hero character;

float speed = 8; //скорость изменения координат
float gravity = -1; // скорость изменения координаты Y

void heroMove(Hero *obj)
{
    obj->y+=obj->dy; //обновление координат
    obj->x+=obj->dx;
    obj->dy-=gravity; // влияние примера гравитации

    obj->currentFrame += 0.2;
    obj->currentTypeAnimation = 0;
    if (obj->currentFrame > 8) obj->currentFrame = 1;

    if (GetKeyState(VK_LEFT)<0 && State==1) // проверка состояния
    {
        obj->currentFrame += 0.2; // подсчет для смены кадра по спрайт
        obj->dx-=speed; // пересылка изменения координаты
        obj->x+=obj->dx; //вычисление конечной координаты
        obj->dx=0; // обнуление для исключение пересчета
        obj->currentTypeAnimation = 1;
        if (obj->currentFrame > 8) obj->currentFrame = 1; //обнуление номера кадра для
    }
    if (GetKeyState(VK_RIGHT)<0 && State==1)
    {
        obj->currentFrame += 0.2;
        obj->dx+=speed;
        obj->x+=obj->dx;
        obj->dx=0;
        obj->currentTypeAnimation = 1;
        if (obj->currentFrame > 8) obj->currentFrame = 1;
    }
    if (GetKeyState(VK_UP)<0 && obj->onGround && State==1)
    {
        obj->dy = speed * -2; // дает импульс для имитации прыжка
        obj->y += obj->dy; //расчет координаты Y
        obj->currentTypeAnimation = 2;
        obj->onGround = false;
    }
}

void characterUpdate(Hero *obj)
{

    heroMove(&character);
    updateCollision(&character, 40);

    renderAreaTexture(&tex_Character, obj->x,obj->y,900,450,112.5,150, character.currentFrame, character.currentTypeAnimation, true);

}

const int H = 15;
const int W = 20;

char TileMap[15][20] = {
"                    ",
"                    ",
"                    ",
"                    ",
"   A                ",
"   A                ",
"   A                ",
"   AAAA             ",
"                  BB",
"                    ",
"            AAA     ",
"                    ",
"                    ",
"BBBBBBBBBBBBBBBBBBBB"
};

void updateCollision(Hero *obj, int tsize){
    if (obj->y < 0) obj->y = 0;
    if (obj->x < 0) obj->x = 0;
    if (obj->y > WindowHeight-190) obj->y = WindowHeight-190;
    if (obj->x > WindowWeight-100) obj->x = WindowWeight-100;

    int objWidth = 90;
    int objHeight = 150;

    for (int i = obj->y/tsize; i < (obj->y+objHeight)/tsize; i++) {
        for (int j = obj->x/tsize; j < (obj->x+objWidth)/tsize; j++) {
            if (TileMap[i][j] != ' ')
            { // check for collision
                    if (obj->y + objHeight < (i+1) * tsize)
                    { // top
                        obj->y = i * tsize - objHeight;
                        if (obj->dy <= 0) obj->onGround = true;
                    }
                    if (obj->y > i*tsize)
                    { // bottom
                        obj->y = (i+1) * tsize;
                    }
                    obj->dy = 0;
            }
        }
    }
    for (int i = obj->y/tsize; i < (obj->y+objHeight)/tsize; i++) {
        for (int j = obj->x/tsize; j < (obj->x+objWidth)/tsize; j++) {
            if (TileMap[i][j] != ' ')
            { // check for collision
                    if (obj->x < j*tsize)
                    { // left
                        obj->x = j * tsize - objWidth;
                    }
                    if (obj->x > j*tsize)
                    { // right
                        obj->x = (j+1) * tsize;
                    }
                    obj->dx = 0;
            }
        }
    }
}

void drawMap(Hero *obj)
{
    float tsize = 40;

    for (int i=0; i<H; i++) //чтение массива по вертикали
    {
        for (int j=0; j<W; j++) // чтение массива по горизонтали
        {
            if (TileMap[i][j]=='A')
            {
                renderAreaTexture(&tex_Blocks,j*tsize,i*tsize,80,40,40,40,0,0,true);
            }
            if (TileMap[i][j]=='B')
            {
                renderAreaTexture(&tex_Blocks,j*tsize,i*tsize,80,40,40,40,1,0,true);
            }
            if (TileMap[i][j]==' ') continue;
        }
    }
}

int WINAPI WinMain(HINSTANCE hInstance,
                   HINSTANCE hPrevInstance,
                   LPSTR lpCmdLine,
                   int nCmdShow)
{
    WNDCLASSEX wcex;
    HWND hwnd;
    HDC hDC;
    HGLRC hRC;
    MSG msg;
    BOOL bQuit = FALSE;

    /* register window class */
    wcex.cbSize = sizeof(WNDCLASSEX);
    wcex.style = CS_OWNDC;
    wcex.lpfnWndProc = WindowProc;
    wcex.cbClsExtra = 0;
    wcex.cbWndExtra = 0;
    wcex.hInstance = hInstance;
    wcex.hIcon = LoadIcon(NULL, IDI_APPLICATION);
    wcex.hCursor = LoadCursor(NULL, IDC_ARROW);
    wcex.hbrBackground = (HBRUSH)GetStockObject(BLACK_BRUSH);
    wcex.lpszMenuName = NULL;
    wcex.lpszClassName = "GLSample";
    wcex.hIconSm = LoadIcon(NULL, IDI_APPLICATION);;


    if (!RegisterClassEx(&wcex))
        return 0;

    /* create main window */
    hwnd = CreateWindowEx(0,
                          "GLSample",
                          "OpenGL Sample",
                          WS_OVERLAPPEDWINDOW,
                          CW_USEDEFAULT,
                          CW_USEDEFAULT,
                          WindowWeight,
                          WindowHeight,
                          NULL,
                          NULL,
                          hInstance,
                          NULL);

    ShowWindow(hwnd, nCmdShow);

    /* enable OpenGL for the window */
    EnableOpenGL(hwnd, &hDC, &hRC);



    RECT rct; //создание переменной с координатами прямоуголника
    GetClientRect(hwnd, &rct); //получение текущих координат окна
    glOrtho(0, rct.right, 0, rct.bottom, 1, -1); //выставляем их как координаты окна

    // create texture (not drawing)
    createTexture(&tex_Character, "content/2DHero.png");
    createTexture(&tex_Background, "content/background.jpg");
    createTexture(&tex_Blocks, "content/SpriteList.png");


    /* program main loop */
    while (!bQuit)
    {
        /* check for messages */
        if (PeekMessage(&msg, NULL, 0, 0, PM_REMOVE))
        {
            /* handle or dispatch messages */
            if (msg.message == WM_QUIT)
            {
                bQuit = TRUE;
            }
            else
            {
                TranslateMessage(&msg);
                DispatchMessage(&msg);
            }
        }
        else
        {
            /* OpenGL animation code goes here */

            glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
            glClear(GL_COLOR_BUFFER_BIT);

            glPushMatrix();
            glRotatef(0.0f, 0.0f, 0.0f, 1.0f);

            glLoadIdentity();
            glOrtho(0, rct.right, rct.bottom, 0, 1, -1); //выставляем их как координаты окна

            switch (State){
            case 0:
                renderFullTexture(&tex_Background, 0,0, WindowWeight, WindowHeight, true);

                buttonRender(&btn[0], "Play", 100, 300, 250, 50, 3);
                buttonRender(&btn[1], "Quit", 100, 360, 150, 50, 3);

                btn[0].visible = TRUE;
                btn[1].visible = TRUE;
                break;
            case 1:
                renderFullTexture(&tex_Background, 0,0, WindowWeight, WindowHeight, true);

                drawMap(&character);
                characterUpdate(&character);

                buttonRender(&btn[2], "Stop game", 250, 50, 250, 50, 3);
                btn[2].visible = TRUE;
                break;
            }

            glPopMatrix();

            SwapBuffers(hDC);
            Sleep(10);
        }
    }

    /* shutdown OpenGL */
    DisableOpenGL(hwnd, hDC, hRC);

    /* destroy the window explicitly */
    DestroyWindow(hwnd);

    return msg.wParam;
}

LRESULT CALLBACK WindowProc(HWND hwnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
    switch (uMsg)
    {
        case WM_CLOSE:
            PostQuitMessage(0);
        break;

        case WM_LBUTTONDOWN:
            for (int i = 0; i < btnCnt; i++)
                if (pointInButton(LOWORD(lParam), HIWORD(lParam), btn[i]))
                {
                    printf("%s\n", btn[i].name);
                    if (strcmp(btn[i].name, "Quit") == 0)
                        PostQuitMessage(0);
                    else if (strcmp(btn[i].name, "Play") == 0)
                    {
                        State = 1;
                    }
                    else if (strcmp(btn[i].name, "Stop game") == 0)
                    {
                        State = 0;
                    }
                }
        break;

        case WM_MOUSEMOVE:
            for (int i = 0; i < btnCnt; i++)
                btn[i].hover = pointInButton(LOWORD(lParam), HIWORD(lParam), btn[i]);
        break;

        case WM_DESTROY:
            return 0;

        case WM_KEYDOWN:
        {
            switch (wParam)
            {
                case VK_ESCAPE:
                    PostQuitMessage(0);
                break;
            }
        }
        break;

                case WM_MOUSEHOVER:


        default:
            return DefWindowProc(hwnd, uMsg, wParam, lParam);
    }

    return 0;
}

void EnableOpenGL(HWND hwnd, HDC* hDC, HGLRC* hRC)
{
    PIXELFORMATDESCRIPTOR pfd;

    int iFormat;

    /* get the device context (DC) */
    *hDC = GetDC(hwnd);

    /* set the pixel format for the DC */
    ZeroMemory(&pfd, sizeof(pfd));

    pfd.nSize = sizeof(pfd);
    pfd.nVersion = 1;
    pfd.dwFlags = PFD_DRAW_TO_WINDOW |
                  PFD_SUPPORT_OPENGL | PFD_DOUBLEBUFFER;
    pfd.iPixelType = PFD_TYPE_RGBA;
    pfd.cColorBits = 24;
    pfd.cDepthBits = 16;
    pfd.iLayerType = PFD_MAIN_PLANE;

    iFormat = ChoosePixelFormat(*hDC, &pfd);

    SetPixelFormat(*hDC, iFormat, &pfd);

    /* create and enable the render context (RC) */
    *hRC = wglCreateContext(*hDC);

    wglMakeCurrent(*hDC, *hRC);
}

void DisableOpenGL (HWND hwnd, HDC hDC, HGLRC hRC)
{
    wglMakeCurrent(NULL, NULL);
    wglDeleteContext(hRC);
    ReleaseDC(hwnd, hDC);
}

