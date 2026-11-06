# Correção de build no Visual Studio 2026

## Ambiente do erro
- Visual Studio 2026 (v18) Community, MSVC 14.51.36231
- Projeto `snes9x`, configuração `Release Unicode Win32`
- Resultado do build: 4 projetos OK (HLSL, OSDependent, glslang, SPIRV), `snes9x` falhou

## Análise do log
O log tem 774 erros, todos em um único arquivo e em apenas 3 linhas-fonte:

| Local | Erros |
|---|---|
| `win32/_tfwopen.h` linhas 90, 92, 95 e 97 | C2065 (`_Openprot` não declarado) e C2039 (`_Openprot` não é membro de `std::ios_base`) |

Cada mensagem aparece 129 vezes (uma por arquivo compilado), porque `_tfwopen.h` é um
*forced include* (`ForcedIncludeFiles` em `snes9xw.vcxproj`). Um único defeito no header
se multiplica por todas as unidades de tradução. Os avisos C4244 do Vulkan são inofensivos.

## Causa raiz
`u8nifstream` (derivada de `std::ifstream`) declarava:

```cpp
int _Prot = (int)ios_base::_Openprot
```

`_Openprot` e o argumento de proteção do construtor de `basic_ifstream` eram detalhes internos
e não padrão da STL da Microsoft. Foram removidos na STL da MSVC 14.51 (VS 2026).
Versões anteriores do VS ainda os tinham, por isso o projeto compilava antes.

## Correção (win32/_tfwopen.h)
Os dois construtores passaram a ter a assinatura padrão:

```cpp
explicit u8nifstream(const char *_Filename, ios_base::openmode _Mode = ios_base::in)
    : std::ifstream(Utf8ToWide(_Filename), _Mode) {}

explicit u8nifstream(const wchar_t *_Filename, ios_base::openmode _Mode = ios_base::in)
    : std::ifstream(_Filename, _Mode) {}
```

Segurança da mudança: o overload UTF-8 já ignorava `_Prot`, e os chamadores
(`wsnes9x.cpp`, `bml.cpp`, `bsx.h`) usam no máximo `(nome, modo)`. Nenhum usa o terceiro argumento.

## Limitações
A correção foi aplicada por análise do log e do código, sem compilar com o MSVC aqui.
Corrigido este erro, outros que estavam escondidos pelo volume do log podem aparecer.
Se aparecerem, envie o novo log.
