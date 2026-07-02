# Refatoracao de Code Smell: too-many-statements

Este documento consolida as ocorrencias de `too-many-statements` identificadas pelo Pylint no projeto e o estado final da refatoracao. O smell R0915 ocorre quando uma funcao ou metodo concentra instrucoes demais, dificultando leitura, manutencao, teste e isolamento de responsabilidades.

## Escopo

As ocorrencias originais foram extraidas de `metrics-before-pylint/pylint_refactor_antes.json`.

| # | Arquivo | Objeto | Mensagem original |
|---|---|---|---|
| 1 | `sample_factory\enjoy.py` | `enjoy` | Too many statements (110/50) |
| 2 | `sample_factory\algo\learning\learner.py` | `Learner._calculate_losses` | Too many statements (76/50) |
| 3 | `sample_factory\algo\learning\learner.py` | `Learner._train` | Too many statements (86/50) |
| 4 | `sample_factory\algo\learning\learner.py` | `Learner._record_summaries` | Too many statements (54/50) |
| 5 | `sample_factory\algo\runners\runner.py` | `Runner.__init__` | Too many statements (57/50) |
| 6 | `sample_factory\cfg\cfg.py` | `add_rl_args` | Too many statements (69/50) |
| 7 | `sample_factory\launcher\run_processes.py` | `run` | Too many statements (67/50) |
| 8 | `sample_factory\launcher\run_slurm.py` | `run_slurm` | Too many statements (56/50) |

## Plano aplicado

Para todas as ocorrencias, a estrategia foi preservar a interface publica e extrair blocos coesos para funcoes ou metodos auxiliares com nomes explicitos. Essa abordagem reduz a contagem de statements nos pontos marcados pelo Pylint sem alterar o contrato dos chamadores.

Riscos observados:

- manter a ordem de execucao original;
- nao alterar argumentos, retornos ou efeitos colaterais;
- preservar atualizacoes de estado compartilhado;
- manter logs, temporizadores e chamadas de subprocessos no mesmo fluxo;
- verificar o resultado com Pylint focado em R0915.

## Ocorrencia 1 - `enjoy`

Arquivo: `sample_factory\enjoy.py`

`enjoy` concentrava preparacao de configuracao, criacao de ambiente, inicializacao do modelo, loop de avaliacao, tratamento de episodios, renderizacao, video e publicacao no HuggingFace.

Refatoracao aplicada:

- `_prepare_eval_config`
- `_init_eval_actor_critic`
- `_init_enjoy_state`
- `_max_frames_reached`
- `_select_actions`
- `_advance_env_once`
- `_handle_env_step_end`
- `_generate_enjoy_outputs`
- `_mean_episode_reward`

Resultado: `enjoy` ficou como orquestrador do fluxo de avaliacao, delegando passos independentes para helpers.

## Ocorrencias 2, 3 e 4 - `Learner`

Arquivo: `sample_factory\algo\learning\learner.py`

Os metodos `_calculate_losses`, `_train` e `_record_summaries` concentravam partes distintas do ciclo de aprendizado: calculo de perdas, loop de epocas/minibatches, otimizacao, KL, early stop e registro de metricas.

Refatoracao aplicada:

- `_calculate_vtrace_advantages`
- `_calculate_advantages_and_returns`
- `_train_loop_initial_state`
- `_get_train_minibatch`
- `_postprocess_losses`
- `_update_recent_kls`
- `_optimizer_step`
- `_after_train_minibatch`
- `_update_early_stop`
- `_record_basic_summaries`
- `_record_advantage_summaries`
- `_record_distribution_summaries`
- `_record_last_batch_summaries`
- `_record_optimizer_summaries`
- `_record_policy_version_summaries`
- `_scalarize_summaries`

Resultado: os metodos principais passaram a coordenar etapas menores do treinamento, mantendo os calculos e atualizacoes de estado no mesmo fluxo logico.

## Ocorrencia 5 - `Runner.__init__`

Arquivo: `sample_factory\algo\runners\runner.py`

`Runner.__init__` inicializava diretamente event loop, estado, estatisticas, writers, handlers, timers e heartbeat.

Refatoracao aplicada:

- `_init_event_loop`
- `_init_state`
- `_init_stats`
- `_init_writers`
- `_init_msg_handlers`
- `_init_timers`

Resultado: o construtor passou a declarar a sequencia de inicializacao e cada helper passou a cuidar de um grupo de atributos relacionado.

## Ocorrencia 6 - `add_rl_args`

Arquivo: `sample_factory\cfg\cfg.py`

`add_rl_args` reunia muitos argumentos de configuracao de RL em uma unica funcao.

Refatoracao aplicada:

- `_add_rl_system_args`
- `_add_rl_regime_args`
- `_add_rl_basic_params`
- `_add_rl_loss_args`
- `_add_rl_policy_gradient_args`
- `_add_rl_optimization_args`
- `_add_rl_learning_rate_args`
- `_add_rl_observation_args`
- `_add_rl_decorrelation_args`
- `_add_rl_performance_args`
- `_add_rl_logging_args`
- `_add_rl_termination_args`
- `_add_rl_checkpoint_args`
- `_add_rl_debug_args`

Resultado: `add_rl_args` ficou apenas como agregador das secoes de argumentos, mantendo a mesma API externa.

## Ocorrencia 7 - `run`

Arquivo: `sample_factory\launcher\run_processes.py`

`run` acumulava preparacao de comandos, selecao de GPU, montagem de ambiente, criacao de subprocessos, coleta de processos finalizados e relatorio de falhas.

Refatoracao aplicada:

- `find_least_busy_gpu`
- `can_squeeze_another_process`
- `prepare_process_command`
- `prepare_process_env`
- `start_experiment_process`
- `collect_finished_processes`
- `report_failed_processes`

Resultado: `run` ficou responsavel pelo loop principal e delegou cada operacao operacional para helpers menores.

## Ocorrencia 8 - `run_slurm`

Arquivo: `sample_factory\launcher\run_slurm.py`

`run_slurm` concentrava criacao de workdir, leitura de template, montagem de `sbatch`, submissao de jobs e escrita de script de cancelamento.

Refatoracao aplicada:

- `ensure_slurm_workdir`
- `load_sbatch_template`
- `slurm_partition_arg`
- `slurm_num_cpus`
- `create_sbatch_files`
- `submit_sbatch_file`
- `submit_sbatch_files`
- `write_scancel_script`

Resultado: `run_slurm` ficou como coordenador do fluxo SLURM, com detalhes de montagem e submissao isolados.

## Verificacao

Comando executado:

```powershell
.venv\Scripts\python.exe -m pylint --disable=all --enable=too-many-statements sample_factory\enjoy.py sample_factory\algo\learning\learner.py sample_factory\algo\runners\runner.py sample_factory\cfg\cfg.py sample_factory\launcher\run_processes.py sample_factory\launcher\run_slurm.py
```

Resultado:

```text
Your code has been rated at 10.00/10
```

O Pylint nao reportou nenhuma ocorrencia de `too-many-statements` nos arquivos verificados.

Observacao: o Pylint tentou gravar cache em `C:\Users\laris\AppData\Local\pylint\pylint\Cache` e recebeu `Permission denied`. Isso nao afetou a analise do smell; foi apenas uma falha de escrita de cache fora do projeto.

## Resultado consolidado

```text
too-many-statements: 8 -> 0 nos arquivos afetados
```
