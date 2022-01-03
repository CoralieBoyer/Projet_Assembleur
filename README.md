; external functions from X11 library
extern XOpenDisplay
extern XDisplayName
extern XCloseDisplay
extern XCreateSimpleWindow
extern XMapWindow
extern XRootWindow
extern XSelectInput
extern XFlush
extern XCreateGC
extern XSetForeground
extern XDrawLine
extern XNextEvent

; external functions from stdio library (ld-linux-x86-64.so.2)    
extern printf
extern exit

%define    StructureNotifyMask    131072
%define KeyPressMask        1
%define ButtonPressMask        4
%define MapNotify        19
%define KeyPress        2
%define ButtonPress        4
%define Expose            12
%define ConfigureNotify        22
%define CreateNotify 16
%define QWORD    8
%define DWORD    4
%define WORD    2
%define BYTE    1

global main

section .data
event:        times    24 dq 0

;on définit la zone que l'on dessine. Ici, la fractale toute entière
x1:    dq    -2.1
x2:    dq    0.6
y1:    dq    -1.2
y2:    dq    1.2

zoom: dd 100
iteration_max: dd 50


x: dd 0.
y: dd 0.

i: dd 0

c_r: dq 0.
c_i: dq 0.
z_r: dq 0.
z_i: dq 0.
float_zero: dq 0.

tmp: dq 0.


section .bss
display_name:    resq    1
screen:            resd    1
depth:             resd    1
connection:        resd    1
width:             resd    1
height:            resd    1
window:        resq    1
gc:        resq    1

image_x: resd 1
image_y: resd 1

section .text
    
;##################################################
;########### PROGRAMME PRINCIPAL ##################
;##################################################

main:
xor     rdi,rdi
call    XOpenDisplay	; Création de display
mov     qword[display_name],rax	; rax=nom du display

; display_name structure
; screen = DefaultScreen(display_name);
mov     rax,qword[display_name]
mov     eax,dword[rax+0xe0]
mov     dword[screen],eax

mov rdi,qword[display_name]
mov esi,dword[screen]
call XRootWindow
mov rbx,rax

mov rdi,qword[display_name]
mov rsi,rbx
mov rdx,10
mov rcx,10
mov r8,400	; largeur
mov r9,400	; hauteur
push 0xFFFFFF	; background  0xRRGGBB
push 0x00FF00
push 1
call XCreateSimpleWindow
mov qword[window],rax

mov rdi,qword[display_name]
mov rsi,qword[window]
mov rdx,131077 ;131072
call XSelectInput

mov rdi,qword[display_name]
mov rsi,qword[window]
call XMapWindow

mov rsi,qword[window]
mov rdx,0
mov rcx,0
call XCreateGC
mov qword[gc],rax

mov rdi,qword[display_name]
mov rsi,qword[gc]
mov rdx,0x000000	; Couleur du crayon
call XSetForeground

boucle: ; boucle de gestion des évènements
mov rdi,qword[display_name]
mov rsi,event
call XNextEvent

cmp dword[event],ConfigureNotify	; à l'apparition de la fenêtre
je dessin							; on saute au label 'dessin'

cmp dword[event],KeyPress			; Si on appuie sur une touche
je closeDisplay						; on saute au label 'closeDisplay' qui ferme la fenêtre
jmp boucle

;#########################################
;#		DEBUT DE LA ZONE DE DESSIN     #
;#########################################
dessin:

;----Calcul taille de l'image----
;image_x = (x2-x1)*zoom
movsd xmm0,[x2]
subsd xmm0,[x1] ; xmm0 <-- x2 - x1
mulsd xmm0,[zoom] ;xmm0 <-- x2-x1 * zoom
cvtsd2si eax,xmm0 ;conversion en int
mov dword[image_x],eax

;image_y = (y2-y1)*zoom
movsd xmm0,[y2]
subsd xmm0,[y1] ; xmm0 <-- y2 - y1
mulsd xmm0,[zoom] ;xmm0 <-- y2-y1 * zoom
cvtsd2si eax,xmm0 ;conversion en int
mov dword[image_y],eax

;----Boucles----
mov dword[x],0
boucleX: ;for (x=0; x<image_x; ++x)
;condition de sortie
mov eax,dword[image_x]
cmp dword[x],eax
jae finBoucleX ;Sortir si x >= image_x

mov dword[y],0
boucleY: ;for (y=0; y<image_y; ++y)
;condition de sortie
mov eax,dword[image_y]
cmp dword[y],eax
jae finBoucleY ;Sortir si y >= image_y

;c_r = x/zoom+x1
mov eax,dword[x]
cvtsi2sd xmm0,eax ;conversion en float sans toucher a la variable
divsd xmm0,[zoom] ;xmm0 <-- x / zoom
addsd xmm0,[x1] ;xmm0 <-- x/zoom + x1
movsd [c_r],xmm0 ;c_r <-- xmm0

;c_i = y/zoom+y1
mov eax,dword[y]
cvtsi2sd xmm0,eax ;conversion en float sans toucher a la variable
divsd xmm0,[zoom] ;xmm0 <-- y / zoom
addsd xmm0,[y1] ;xmm0 <-- y/zoom + y1
movsd [c_i],xmm0 ; c_i <-- xmm0

;z_r = 0
movsd xmm0, [float_zero] ;xmm0 <-- 0.
movsd [z_r],xmm0 ;z_r <-- 0.

;z_i = 0
movsd xmm0, [float_zero] ;xmm0 <-- 0.
movsd [z_i],xmm0 ;z_i <-- 0.

;i = 0
mov dword[i],0

doWhile:
;tmp=z_r
movsd xmm0,[z_r]
movsd [tmp],xmm0

;z_r=z_r*z_r-z_i*z_i+c_r
mulsd xmm0,xmm0 ;xmm0 <-- z_r * z_r
movsd xmm1,[z_i]
mulsd xmm1,xmm1 ; xmm1 <-- z_i * z_i
subsd xmm0,xmm1 ; xmm0 <-- z_r*z_r - z_i*z_i
addsd xmm0,[c_r] ;xmm0 <-- z_r*z_r-z_i*z_i + c_r
movsd [z_r],xmm0 ;z_r <-- xmm0

;z_i = 2*z_i*tmp+c_i
movsd xmm0,[z_i]
addsd xmm0,xmm0 ;xmm0 <-- 2 * z_i
mulsd xmm0,[tmp] ; xmm0 <-- 2*z_i * tmp
addsd xmm0,[c_i] ; xmm0 <-- 2*z_i*tmp + c_i
movsd [z_i],xmm0 ;z_i <-- xmm0

;i=i+1
inc dword[i]

;tant que z_r*z_r+z_i*z_i < 4
movsd xmm0,[z_r]
mulsd xmm0,xmm0 ;xmm0 <-- z_r * z_r
movsd xmm1,[z_i]
mulsd xmm1,xmm1 ; xmm1 <-- z_i * z_i
addsd xmm0,xmm1 ; xmm0 <-- z_r*z_r + z_i*z_i
cvtsd2si rax,xmm0 ; conversion en int
cmp rax,4
jge finDoWhile ;fin doWhile si z_r*z_r+z_i*z_i >= 4

;et i < iteration_max
mov eax,dword[i]
cmp eax,dword[iteration_max]
jb doWhile ;recommencer la boucle si i<iteration_max

finDoWhile:

;si i = iteration_max
cmp eax,dword[iteration_max]
jne pasDessin ;sauter le dessin si i != iteration_max

;dessin
mov rdi,qword[display_name]
mov rsi,qword[window]
mov rdx,qword[gc]
mov ecx,dword[x]	; coordonnée source en x
mov r8d,dword[y]	; coordonnée source en y
mov r9d,dword[x]	; coordonnée destination en x
push qword[y]		; coordonnée destination en y
call XDrawLine

pasDessin:

;instructions fin de boucle Y
inc dword[y] ;++y
jmp boucleY
finBoucleY:

;instructions fin de boucle x
inc dword[x] ;++x
jmp boucleX
finBoucleX:

; ############################
; # FIN DE LA ZONE DE DESSIN #
; ############################
jmp flush

flush:
mov rdi,qword[display_name]
call XFlush
jmp boucle
mov rax,34
syscall

closeDisplay:
    mov     rax,qword[display_name]
    mov     rdi,rax
    call    XCloseDisplay
    xor	    rdi,rdi
    call    exit
