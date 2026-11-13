# SOC Lab com Wazuh: FIM, PowerShell e regras customizadas

Laboratório de monitoramento de endpoint com Wazuh, usado para praticar detecção e triagem de eventos em um ambiente Windows.

## Arquitetura

- **Servidor Wazuh:** Ubuntu Server 24.04.5 LTS, com Wazuh 4.9.2 (indexer, manager e dashboard), em VM no VMware. Disco expandido para 48 GB.
- **Endpoint monitorado:** Windows 11 Pro com agente Wazuh 4.9.2, nome `WIN-SOC-LAB`.
- **Atacante simulado:** Kali Linux, na mesma rede.

## Configuração

1. **Agente Windows:** monitoramento de arquivos (FIM) em tempo real na pasta `C:\Teste-FIM`, e leitura do canal `Microsoft-Windows-PowerShell/Operational`.
2. **Registro de blocos de script:** ativado pela chave `ScriptBlockLogging`, para gerar o evento 4104.
3. **Testes:** Atomic Red Team, com a pasta `C:\AtomicRedTeam` excluída da proteção do Microsoft Defender. A exclusão foi removida ao final.
4. **Regras customizadas:** arquivo `/var/ossec/etc/rules/local_rules.xml`.

## Cenários testados

| Cenário | Técnica MITRE | Ação executada | Regra disparada | Nível | Resultado |
|---|---|---|---|---|---|
| Arquivo criado em pasta monitorada | T1074 | `New-Item` em `C:\Teste-FIM` | 554 (padrão) e 100004 | 5 e 8 | Detectado |
| Arquivo alterado | FIM | `Add-Content` | 550 | 7 | Detectado |
| Arquivo removido | FIM | `Remove-Item` | 553 | 7 | Detectado |
| PowerShell com comando codificado | T1059.001 (Atomic 17) | `powershell.exe -e ...` | 100003 | 6 | Detectado, mas a regra é ampla |
| Persistência por chave Run | T1547.001 (Atomic 1) | `Reg Key Run` | 100002 | 10 | **Não detectado** |

## Regras customizadas

- **100002:** alteração na chave `CurrentVersion\Run`, mapeada a T1547.001. Não gerou alerta nos testes.
- **100003:** qualquer evento 4104 do PowerShell, mapeado a T1059.001. Gerou 102 alertas em um dia, incluindo atividade normal do sistema. Ela precisa de filtro pelo conteúdo do bloco de script.
- **100004:** criação de arquivo na pasta `Teste-FIM`, mapeada a T1074.

## Limitações

- O ambiente é um laboratório isolado, sem integração com outros sistemas.
- A regra 100003 é ampla e gera falsos positivos. Em produção, precisaria ser refinada.
- O teste de persistência por Run key não gerou alerta. A causa ainda não foi identificada: o FIM foi configurado para `HKLM` e `HKCU`, mas o evento não chegou ao Dashboard.
- O Suricata e o relatório de resposta a incidente ficaram para uma próxima versão.

## Aprendizados

- Regras customizadas precisam de IDs únicos e de validação com `wazuh-logtest`.
- Um evento que chega ao SIEM não gera alerta sem uma regra correspondente.
- Antivírus pode bloquear ferramentas de simulação de ataque, o que exige exclusões controladas e removidas ao final.
