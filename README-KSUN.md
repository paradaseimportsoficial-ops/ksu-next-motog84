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

Foi recebido outro boot, com identificação interna `V1TCS35H.88-16-1` / bangkk / d09472 e kernel `5.4.284-moto-gef07def7b0a7`. SHA256: `4b5b3930abb5ccfbc2a516bc9b7c613ff22e31ba6b6f0dfdabe2dfe07776252f`. A assinatura/procedência Motorola não foi autenticada.

Foi montada uma imagem de engenharia unsigned de 44.187.648 bytes, preservando o ramdisk byte a byte, o cabeçalho v3 e seus campos, exceto kernel_size. O kernel foi substituído pelo Image compilado. Os metadados AVB originais não foram copiados porque descrevem o kernel anterior. SHA256 da imagem: `029c521572f57cf2be44b6d03f33ada380d5212c716652d6f13d3d37fdb268ba`. O pacote `MotoG84-KSUN-boot-88-16-1-EXPERIMENTAL.zip` contém a imagem, script reproduzível, relatório e checksums; não é instalador TWRP. SHA256 do ZIP: `e9c3e9a26676f7c185eed3313e72bca1fe6103f92a8d8cf19894b99c7cec74e2`.

Verificados: round-trip do payload original, hashes dos insumos e de todos os arquivos do manifesto de compilação, estrutura e alinhamento da saída, preservação exata do ramdisk, rejeição de entradas truncadas/cabeçalho incompatível. Há oito diferenças CONFIG entre este kernel original e a build, incluindo KSU, VDSO, RELR e localversion. O ramdisk não contém arquivos .ko. Faltam vendor_boot e módulos externos da mesma base para conferir compatibilidade. Nenhum boot, root, hardware ou downgrade foi testado. Não tratar a imagem como validada para a base atual 88-16-9.

O boot original analisado é de uma base diferente: `5.4.295-moto-17290-g91c2dab5855a`, compilação do aparelho `V1TCS35H.88-16-9`. SHA-256 do boot enviado: `f214a2339f0914e0d0871012dd5d82bdd546a7cdff5149f25a37f9c95074f0a0`. A imagem tem cabeçalho Android boot v3 e configuração embutida; ela não é usada automaticamente para empacotar o kernel antigo.

O pacote de compilação **não é boot.img nem instalador TWRP**. Ainda faltam validar módulos externos/vendor, DTB/DTBO, firmware e Manager compatível, além de teste no dispositivo. Não contém instruções de downgrade nem modifica bootloader/GPT. Não faça downgrade ou flash deste artefato somente porque a compilação passou.


## Vendor boot recebido — incompatibilidade encontrada

O vendor_boot recebido posteriormente tem SHA256 `04fcd581c15a5d52bbb2335f39592fb1e21b9733945f88ff9aeb74eadec09559`, cabeçalho v3, página 4096, vendor ramdisk gzip de 731250 bytes e DTB de 323929 bytes. A cmdline identifica bangkk. Os 48 módulos ELF64 ARM64 contidos no ramdisk declaram vermagic `5.4.295-moto-17286-gd5c26e98f35d SMP preempt mod_unload modversions aarch64`, diferente do boot base recebido (5.4.284) e da build.

A comparação das entradas ELF `__versions` desses módulos com o `Module.symvers` da build encontrou CRCs divergentes em todos os 48 módulos. Exemplo: `module_layout` exige `0x1e5b7ab7`, mas a build exporta `0x5661dded`. Portanto este conjunto de módulos não está compatível com a build experimental; não montar um instalador usando esse vendor_boot. Não se corrigiu vermagic nem se desabilitou a checagem de versões. Alguns drivers podem estar embutidos na build; isso não valida os demais módulos externos.

Próximo insumo: vendor_boot extraído do mesmo pacote de firmware V1TCS35H.88-16-1 do boot de base, além dos módulos vendor correspondentes. A versão exata do firmware deste vendor_boot não foi determinada só pelo vermagic. A imagem de engenharia entregue continua sem validação para instalação.


## Vendor boot corrigido recebido

O usuário confirmou ter enviado anteriormente um vendor_boot de outra versão e forneceu outro arquivo. SHA256 do novo: `f9ce2c8952486714f3e30afc4a43d90206ad25c40fcd325312a5ae09bf593d42`; tamanho 100663296 bytes; cabeçalho v3; página 4096; ramdisk gzip 731078 bytes.

Os 48 módulos agora declaram exatamente a versão do kernel do boot base: `5.4.284-moto-gef07def7b0a7 SMP preempt mod_unload modversions aarch64`. Isso resolve a divergência de versão do insumo anterior, mas não a ABI da compilação experimental: a comparação das seções ELF __versions com o Module.symvers da build ainda aponta CRCs divergentes em todos os 48 módulos; um módulo também requer símbolos não presentes nesse Module.symvers. Exemplo verificado em _qcom_scm.ko: module_layout `0x1e5b7ab7` versus build `0x5661dded`; of_find_property `0x03d022de` versus build `0x43bd736f`.

A revisão do patch gerado confirma que a integração KSU acrescenta atomic_t filter_count a struct seccomp em include/linux/seccomp.h. Esta é uma alteração relevante a investigar junto às diferenças VDSO/configuração/fonte; a causa individual dos CRCs divergentes ainda não foi isolada. Não contornar o problema alterando apenas strings vermagic ou CRCs. Próxima etapa técnica: reproduzir e comparar uma build sem KSU e com a configuração original, para separar diferenças da base/toolchain das introduzidas pela integração. Depois ajustar/recompilar os módulos necessários ou preservar a ABI de forma comprovada. O novo vendor_boot não foi modificado, e nenhum pacote de instalação foi gerado a partir dele.


## R2 — seccomp preservado e dependências entre módulos verificadas

A execução 41383122995 compilou kernel e KernelSU com sucesso, preservando include/linux/seccomp.h. A verificação automatizada ficou vermelha por tratar duas dependências externas de mmi_info como símbolos obrigatórios do kernel. O resultado bruto foi zero CRCs divergentes nos três módulos Motorola.

A análise local do vendor_boot correto confirmou os exports ELF absolutos de mmi_annotate.ko: mmi_annotate=0xf3cb0240 e mmi_annotate_persist=0x5482805e. Ambos coincidem com __versions de mmi_info.ko; modules.dep declara a dependência e modules.load coloca o provedor antes do consumidor. Resolvidas essas dependências, os checks foram 35/35, 42/42 e 21/21 para mmi_annotate, mmi_info e qpnp_adaptive_charge, respectivamente. O workflow foi corrigido para validar o provedor antes de resolver seus exports e passou a ser manual para evitar recompilações devidas a ajustes de relatório. Não foi alegado sucesso retroativo da execução vermelha.

Foi gerado localmente boot v3 unsigned R2, preservando integralmente o ramdisk 88-16-1 e o cabeçalho exceto kernel_size. Kernel 5.4.284-moto-dirty; SHA256 do boot: 2a987b08df6a59e7388bf8d7cdf4cedcf21b17932dd21d59ee4f9286e77d105c. Pacote MotoG84-KSUN-88-16-1-R2-EXPERIMENTAL.zip, 26329860 bytes, SHA256 4df10b352a164cf2a8be1369a56e13588b73e417bfff91e9c14b196686e674c3. Digest do artefato GitHub foi conferido antes do empacotamento; os hashes internos do ZIP final também passaram.

A verificação acima é parcial: não demonstra boot, root, funcionamento do Manager, DTBO nem compatibilidade de todos os módulos vendor. Os 45 módulos no diretório 5.4-gki continuam divergentes como na referência sem KSU, e seu uso no aparelho precisa ser confirmado. Firmware atual informada anteriormente era 88-16-9; R2 não é validado para ela. ZIP não é instalador TWRP. Nenhum flash/downgrade foi feito.


## Alvo confirmado: manter firmware 88-16-9

O usuário confirmou que o aparelho permanece em V1TCS35H.88-16-9. O boot original foi recuperado e reconferido: SHA256 f214a2339f0914e0d0871012dd5d82bdd546a7cdff5149f25a37f9c95074f0a0, Linux 5.4.295-moto-17290-g91c2dab5855a, Clang 11.0.2 r383902b1. R2/5.4.284 não foi promovido a candidato validado para esse sistema.

Nova consulta ao repositório oficial: commit 91c2dab5855a não encontrado (API 422); matching-refs para tags MMI-V1TCS35H retorna somente MMI-V1TCS35H.88-16-1. Foi também inspecionada a tag MMI-U1TCS34.22-64-19-4-4, citada em solicitação pública como fonte de um aparelho 5.4.295: o commit eb4e9e3927e1e9943ecc9296dbdaaca351fcd0ba declara SUBLEVEL=274 no Makefile, portanto não foi adotado como fonte exata de 88-16-9.

Pendências para manter firmware atual: obter/identificar fonte compatível e comparar vendor_boot e módulos da própria 88-16-9. O primeiro vendor_boot enviado declarava g d5c26e98f35d (sem o espaço), diferente de g91c2dab5855a, e foi confirmado pelo usuário como sendo de outra versão; o segundo corresponde à base antiga 88-16-1. Nenhuma compilação nova para 88-16-9 foi iniciada sem resolver a base. Não há orientação de downgrade nem solicitação publicada a terceiros.
