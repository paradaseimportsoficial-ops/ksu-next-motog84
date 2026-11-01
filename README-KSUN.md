# Moto G84 — KernelSU Next experimental

Workflow: `.github/workflows/ksun-bangkk-experimental.yml`.

## Objetivo

Compilar Linux 5.4.284 da Motorola para bangkk com KernelSU Next integrado. O workflow antigo `build.yml` é de APatch e foi preservado; use o workflow **Moto G84 - KernelSU Next experimental**.

## Fontes fixadas

- Motorola `MMI-V1TCS35H.88-16-1`, commit `9b862eda32365382ce5beebc8e616db1507eed1e`.
- KernelSU Next oficial, branch `legacy`, commit `b933b1f728116a456fbebf27b6a8772a640cdf23` (2026-10-28). Usa o modo syscall-table para kernels anteriores a 5.10. Não são os binários GKI v3.4.2.
- Android Clang `r383902b1` (11.0.2), repositório de prebuilts no commit `7e2d7f3e8dba20760df041bf90ca5a15913631c8`.
- Configuração montada com os fragmentos oficiais holi-qgki/user e moto-holi-bangkk da mesma tag; sem fallback para defconfig genérico.

## Execução

Em Actions, selecione **Moto G84 - KernelSU Next experimental**, depois **Run workflow**. A compilação completa é iniciada manualmente para evitar recompilar ao ajustar documentação ou empacotamento. O driver KernelSU é compilado primeiro para detectar incompatibilidades antes de compilar todo o kernel. Erros em qualquer comando interrompem a etapa; os logs são enviados mesmo em caso de falha.

Se a compilação terminar com sucesso, o artefato contém `Image`, módulos internos, configuração, `Module.symvers`, versões fixadas, patches gerados durante a build e SHA256SUMS. Os logs registram o compilador, integração e resultado.

## Estado e limites

A compilação completa passou na [execução 41270487801](https://github.com/paradaseimportsoficial-ops/ksu-next-motog84/actions/runs/41270487801). O driver foi compilado primeiro, seguido por Image e módulos. Foi necessário gerar previamente os cabeçalhos do SELinux para o teste isolado do driver. Mesmo uma build verde não comprova boot, root ou funcionamento do hardware.

O [empacotamento 41317916208](https://github.com/paradaseimportsoficial-ops/ksu-next-motog84/actions/runs/41317916208) verificou os 30 arquivos do manifesto de checksums e gerou um ZIP de 17.799.977 bytes. O pacote inicial de 2,36 GB havia incluído árvores de fonte/build pelos links criados por modules_install; o workflow foi corrigido para excluir esses links. O ZIP compacto contém Image, 9 arquivos .ko, metadados de módulos, configuração, Module.symvers e os patches de integração. Não contém instalador.

Resultado validado:
- Kernel: `5.4.284-Tecnico-Root-KSUN-test+`.
- `CONFIG_KSU=y` e `CONFIG_KSU_SYSCALL_TABLE_HOOK=y`.
- Clang Android 11.0.2 / r383902b1.
- SHA-256 do ZIP compacto: `3d995a73b2a53c83970a905d729b4c5b19e8955047df8259483d7f73c53d0197`.
- Todos os hashes internos foram conferidos também após baixar o artefato.

A comparação preliminar entre a configuração da build e a extraída do boot atual apontou 14 diferenças de valores CONFIG e 13 opções `=m` em cada configuração. Isso não verifica ABI/CRCs nem garante compatibilidade com módulos externos. A extração de símbolos iniciada para essa comparação ainda não produziu um relatório de ABI validado.

Próxima dependência: obter e validar o boot original da base `V1TCS35H.88-16-1` antes de preparar uma imagem para essa base. O boot fornecido de `88-16-9` não foi automaticamente combinado com o kernel antigo. Nenhum downgrade foi executado ou validado.

O boot original analisado é de uma base diferente: `5.4.295-moto-17290-g91c2dab5855a`, compilação do aparelho `V1TCS35H.88-16-9`. SHA-256 do boot enviado: `f214a2339f0914e0d0871012dd5d82bdd546a7cdff5149f25a37f9c95074f0a0`. A imagem tem cabeçalho Android boot v3 e configuração embutida; ela não é usada automaticamente para empacotar o kernel antigo.

O pacote de compilação **não é boot.img nem instalador TWRP**. Ainda faltam validar módulos externos/vendor, DTB/DTBO, firmware e Manager compatível, além de teste no dispositivo. Não contém instruções de downgrade nem modifica bootloader/GPT. Não faça downgrade ou flash deste artefato somente porque a compilação passou.
