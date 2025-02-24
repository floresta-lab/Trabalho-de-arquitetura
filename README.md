# Trabalho-de-arquitetura
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

const char* hexa_to_bin(char c) {
    switch (c) {
        case '0': return "0000";
        case '1': return "0001";
        case '2': return "0010";
        case '3': return "0011";
        case '4': return "0100";
        case '5': return "0101";
        case '6': return "0110";
        case '7': return "0111";
        case '8': return "1000";
        case '9': return "1001";
        case 'A': case 'a': return "1010";
        case 'B': case 'b': return "1011";
        case 'C': case 'c': return "1100";
        case 'D': case 'd': return "1101";
        case 'E': case 'e': return "1110";
        case 'F': case 'f': return "1111";
        default: return "";
    }
}

void hex_to_bin(const char* hex, char* bin) {
    bin[0] = '\0'; // Inicializa a string binária
    while (*hex) {
        strcat(bin, hexa_to_bin(*hex++));
    }
}

int bin_to_int(const char* bin, int start, int length) {
    int value = 0;
    for (int i = 0; i < length; i++) {
        value = (value << 1) | (bin[start + i] - '0');
    }
    return value;
}

void print_registers(int* R, int PC, int SP, int* FLAGS) {
    // Função para imprimir o conteúdo dos registradores
    printf("Conteúdo dos registradores:\n");
    for (int i = 0; i < 8; i++) {
        printf("R%d: 0x%04x\n", i, R[i]);
    }
    printf("PC: 0x%04x\nSP: 0x%04x\nFLAGS: C=%d Ov=%d Z=%d S=%d\n", PC, SP, FLAGS[0], FLAGS[1], FLAGS[2], FLAGS[3]);
}

void print_instruction(const char* bin, int* R, int* FLAGS, int* SP, int* memoria, FILE* pilha_file, int* PC) {
    int opcode = bin_to_int(bin, 0, 4);
    int rd = bin_to_int(bin, 4, 3);
    int rm = bin_to_int(bin, 7, 3);
    int rn = bin_to_int(bin, 10, 3);
    int im = bin_to_int(bin, 13, 3);

    switch (opcode) {
        case 0x1: // MOV
            if (bin[4] == '0') {
                printf("MOV R%d, R%d\n", rd, rm);
                R[rd] = R[rm];
            } else {
                printf("MOV R%d, #%d\n", rd, im);
                R[rd] = im;
            }
            break;
        case 0x4: // ADD
            printf("ADD R%d, R%d, R%d\n", rd, rm, rn);
            R[rd] = R[rm] + R[rn];
            break;
        case 0x5: // SUB
            printf("SUB R%d, R%d, R%d\n", rd, rm, rn);
            R[rd] = R[rm] - R[rn];
            break;
        case 0x0: // Comandos especiais e NOP
            if (strcmp(bin, "0000000000000000") == 0) {
                printf("NOP\n");
                // Imprime o conteúdo dos registradores
                print_registers(R, *PC, *SP, FLAGS);
            } else if (strcmp(bin, "0000000000001001") == 0) { // PSH
                printf("PSH R%d\n", rm);
                memoria[--(*SP)] = R[rm];
                fprintf(pilha_file, "0x%04x: 0x%04x\n", *SP, R[rm]);
            } else if (strcmp(bin, "0000011100000010") == 0) { // POP
                printf("POP R%d\n", rd);
                R[rd] = memoria[(*SP)++];
            } else if (strcmp(bin, "0000000001011111") == 0) { // CMP
                printf("CMP R%d, R%d\n", rm, rn);
                FLAGS[2] = (R[rm] == R[rn]) ? 1 : 0; // Z flag
                FLAGS[3] = (R[rm] < R[rn]) ? 1 : 0;  // S flag
            } else if (strncmp(bin, "0000100", 7) == 0) { // JMP
                printf("JMP #%d\n", im);
                *PC += im;
            } else if (strcmp(bin, "0000111111000001") == 0) { // JEQ
                printf("JEQ Inicio\n");
                if (FLAGS[2] == 1) { // Z flag
                    *PC = 0;
                }
            } else {
                printf("Instrução não reconhecida: %s\n", bin);
            }
            break;
        case 0xF: // HALT
            printf("HALT\n");
            break;
        default:
            printf("Instrução não reconhecida: %s\n", bin);
    }
}

int main() {
    char linha[100], bin[100];
    int pc = 0x0000, sp = 0x8200, R[8] = {0}, FLAGS[4] = {0}, memoria[65536] = {0};
    const char *caminho_cod = "C:\\Users\\Dougl\\OneDrive\\Documentos\\trabalho de arq2\\cod.txt";
    const char *caminho_memoria = "C:\\Users\\Dougl\\OneDrive\\Documentos\\trabalho de arq2\\memoria.txt";
    const char *caminho_pilha = "C:\\Users\\Dougl\\OneDrive\\Documentos\\trabalho de arq2\\pilha.txt";
    FILE *pot = fopen(caminho_cod, "r");
    if (!pot) {
        perror("Erro ao abrir o arquivo de código");
        return 1;
    }

    FILE *memoria_file = fopen(caminho_memoria, "w");
    if (!memoria_file) {
        perror("Erro ao abrir o arquivo de memória");
        fclose(pot);
        return 1;
    }

    FILE *pilha_file = fopen(caminho_pilha, "w");
    if (!pilha_file) {
        perror("Erro ao abrir o arquivo de pilha");
        fclose(pot);
        fclose(memoria_file);
        return 1;
    }

    while (fgets(linha, sizeof(linha), pot)) {
        int endereco;
        char conteudo_hex[5];
        if (sscanf(linha, "%x: 0x%4s", &endereco, conteudo_hex) == 2) {
            strcpy(bin, "");
            hex_to_bin(conteudo_hex, bin);
            fprintf(memoria_file, "%.4x: %s\n", pc, bin);
            fprintf(pilha_file, "%.4x: %s\n", sp, bin);
            print_instruction(bin, R, FLAGS, &sp, memoria, pilha_file, &pc);
            pc += 2;
            if (bin_to_int(bin, 0, 4) == 0xF) break;
        }
    }

    fclose(pot);
    fclose(memoria_file);
    fclose(pilha_file);

    // Imprime o conteúdo dos registradores
    print_registers(R, pc, sp, FLAGS);

    // Abre o arquivo de memória para escrita adicional
    memoria_file = fopen(caminho_memoria, "a");
    if (!memoria_file) {
        perror("Erro ao abrir memoria.txt");
        return 1;
    }

    // Imprime o conteúdo da memória
    fprintf(memoria_file, "\nConteúdo da memória:\n");
    for (int i = 0; i < 65536; i++) { // Corrigido aqui
        if (memoria[i] != 0) {
            fprintf(memoria_file, "0x%04x: 0x%04x\n", i, memoria[i]);
        }
    }

    // Abre o arquivo da pilha para escrita adicional
    pilha_file = fopen(caminho_pilha, "a");
    if (!pilha_file) {
        perror("Erro ao abrir pilha.txt");
        fclose(memoria_file);
        return 1;
    }

    // Imprime o conteúdo da pilha
fprintf(pilha_file, "\nConteúdo da pilha:\n");
for (int i = sp; i < 0x8210; i++) {
    if (memoria[i] != 0) {
        fprintf(pilha_file, "0x%04x: 0x%04x\n", i, memoria[i]);
    }
}

fclose(memoria_file);
fclose(pilha_file);

return 0;
}
