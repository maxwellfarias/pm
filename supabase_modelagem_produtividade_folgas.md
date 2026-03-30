# Documentacao SQL - Supabase

Este documento contem uma sugestao de modelagem em PostgreSQL/Supabase para implementar:

1. Cadastro de militares
2. Controle de produtividade
3. Concessao de folga
4. Consulta de militares com direito a folga por produtividade

## 1. Premissas adotadas

Como alguns campos necessarios para as regras de negocio nao estavam explicitamente no cadastro inicial, a modelagem abaixo adiciona:

1. `data_nascimento`: necessaria para folga de aniversario.
2. `saldo_banco_horas_minutos`: necessaria para controlar folgas por banco de horas.
3. `saldo_pontos`: saldo atual de pontos disponiveis para folga por produtividade.
4. `nome_guerra_snapshot` no registro de produtividade: preserva o nome de guerra usado na data do registro, mesmo que ele seja alterado depois.

Observacoes importantes:

1. Cada registro de produtividade representa `1 militar + 1 tipo de ocorrencia`.
2. Se a mesma ocorrencia envolver varios militares, basta criar varios registros com o mesmo `numero_ocorrencia`.
3. Se a mesma ocorrencia tiver mais de um tipo produtivo (ex.: drogas e arma de fogo), crie um registro para cada tipo.
4. Quando uma folga for cadastrada com motivo `produtividade`, o sistema desconta automaticamente `300 pontos`.
5. Quando uma folga for cadastrada com motivo `banco_horas`, o sistema desconta automaticamente o saldo informado em `horas_utilizadas_minutos`.

## 2. Script SQL completo

Cole o bloco abaixo no SQL Editor do Supabase.

```sql
create extension if not exists pgcrypto;

do $$
begin
  if not exists (
    select 1
    from pg_type
    where typname = 'graduacao_enum'
  ) then
    create type public.graduacao_enum as enum (
      'soldado',
      'cabo',
      '3_sargento',
      '2_sargento',
      '1_sargento',
      'subtenente',
      'aspirante_a_oficial',
      '2_tenente',
      '1_tenente',
      'capitao',
      'major',
      'tenente_coronel',
      'coronel'
    );
  end if;
end
$$;

do $$
begin
  if not exists (
    select 1
    from pg_type
    where typname = 'tipo_ocorrencia_enum'
  ) then
    create type public.tipo_ocorrencia_enum as enum (
      'arma_de_fogo',
      'drogas',
      'prisao_homicida',
      'TCO',
      'apreensao_veiculo'
    );
  end if;
end
$$;

do $$
begin
  if not exists (
    select 1
    from pg_type
    where typname = 'motivo_folga_enum'
  ) then
    create type public.motivo_folga_enum as enum (
      'banco_horas',
      'produtividade',
      'aniversario'
    );
  end if;
end
$$;

do $$
begin
  if not exists (
    select 1
    from pg_type
    where typname = 'status_folga_enum'
  ) then
    create type public.status_folga_enum as enum (
      'pendente',
      'concluida'
    );
  end if;
end
$$;

create table if not exists public.militares (
  id uuid primary key default gen_random_uuid(),
  matricula varchar(30) not null unique,
  cpf char(11) not null unique,
  nome_completo text not null,
  nome_guerra text not null,
  graduacao public.graduacao_enum not null,
  data_nascimento date not null,
  saldo_pontos integer not null default 0 check (saldo_pontos >= 0),
  saldo_banco_horas_minutos integer not null default 0 check (saldo_banco_horas_minutos >= 0),
  ativo boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  constraint ck_militares_cpf_formato check (cpf ~ '^[0-9]{11}$')
);

create table if not exists public.registros_produtividade (
  id uuid primary key default gen_random_uuid(),
  numero_ocorrencia varchar(50) not null,
  militar_id uuid not null references public.militares(id) on update cascade on delete restrict,
  nome_guerra_snapshot text not null,
  tipo_ocorrencia public.tipo_ocorrencia_enum not null,
  quantidade_drogas_gramas integer,
  pontos_creditados integer not null check (pontos_creditados in (75, 150, 300)),
  detalhes text,
  data_hora_ocorrencia timestamptz not null,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  constraint uq_registros_produtividade unique (numero_ocorrencia, militar_id, tipo_ocorrencia),
  constraint ck_registros_produtividade_drogas check (
    (
      tipo_ocorrencia = 'drogas'
      and quantidade_drogas_gramas is not null
      and quantidade_drogas_gramas >= 0
    )
    or
    (
      tipo_ocorrencia <> 'drogas'
      and quantidade_drogas_gramas is null
    )
  )
);

create table if not exists public.folgas (
  id uuid primary key default gen_random_uuid(),
  militar_id uuid not null references public.militares(id) on update cascade on delete restrict,
  data_folga date not null,
  motivo public.motivo_folga_enum not null,
  status public.status_folga_enum not null default 'pendente',
  horas_utilizadas_minutos integer,
  pontos_descontados integer not null default 0 check (pontos_descontados in (0, 300)),
  observacao text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  constraint uq_folga_por_dia unique (militar_id, data_folga),
  constraint ck_folgas_horas_utilizadas check (
    horas_utilizadas_minutos is null or horas_utilizadas_minutos > 0
  )
);

create index if not exists idx_registros_produtividade_militar_data
  on public.registros_produtividade (militar_id, data_hora_ocorrencia desc);

create index if not exists idx_registros_produtividade_numero
  on public.registros_produtividade (numero_ocorrencia);

create index if not exists idx_folgas_militar_data
  on public.folgas (militar_id, data_folga desc);

create or replace function public.fn_set_updated_at()
returns trigger
language plpgsql
as $$
begin
  new.updated_at := now();
  return new;
end;
$$;

create or replace function public.fn_calcular_pontos_ocorrencia(
  p_tipo public.tipo_ocorrencia_enum,
  p_quantidade_drogas_gramas integer default null
)
returns integer
language plpgsql
immutable
as $$
begin
  if p_tipo = 'arma_de_fogo' then
    return 300;
  end if;

  if p_tipo = 'prisao_homicida' then
    return 75;
  end if;

  if p_tipo = 'TCO' then
    return 0;
  end if;

  if p_tipo = 'apreensao_veiculo' then
    return 75;
  end if;

  if p_tipo = 'drogas' then
    if p_quantidade_drogas_gramas is null then
      raise exception 'Informe a quantidade de drogas em gramas.';
    end if;

    if p_quantidade_drogas_gramas between 0 and 499 then
      return 75;
    elsif p_quantidade_drogas_gramas between 500 and 999 then
      return 150;
    else
      return 300;
    end if;
  end if;

  raise exception 'Tipo de ocorrencia invalido.';
end;
$$;

create or replace function public.fn_preparar_registro_produtividade()
returns trigger
language plpgsql
as $$
declare
  v_nome_guerra text;
begin
  select nome_guerra
    into v_nome_guerra
  from public.militares
  where id = new.militar_id;

  if not found then
    raise exception 'Militar nao encontrado.';
  end if;

  new.nome_guerra_snapshot := v_nome_guerra;
  new.pontos_creditados := public.fn_calcular_pontos_ocorrencia(
    new.tipo_ocorrencia,
    new.quantidade_drogas_gramas
  );

  return new;
end;
$$;

create or replace function public.fn_sincronizar_pontos_produtividade()
returns trigger
language plpgsql
as $$
begin
  if tg_op = 'INSERT' then
    update public.militares
       set saldo_pontos = saldo_pontos + new.pontos_creditados
     where id = new.militar_id;

    return new;
  end if;

  if tg_op = 'DELETE' then
    update public.militares
       set saldo_pontos = saldo_pontos - old.pontos_creditados
     where id = old.militar_id;

    return old;
  end if;

  if tg_op = 'UPDATE' then
    if old.militar_id = new.militar_id then
      update public.militares
         set saldo_pontos = saldo_pontos + new.pontos_creditados - old.pontos_creditados
       where id = new.militar_id;
    else
      update public.militares
         set saldo_pontos = saldo_pontos - old.pontos_creditados
       where id = old.militar_id;

      update public.militares
         set saldo_pontos = saldo_pontos + new.pontos_creditados
       where id = new.militar_id;
    end if;

    return new;
  end if;

  return null;
end;
$$;

create or replace function public.fn_preparar_folga()
returns trigger
language plpgsql
as $$
declare
  v_data_nascimento date;
  v_saldo_pontos integer;
  v_saldo_banco_horas integer;
  v_pontos_disponiveis integer;
  v_horas_disponiveis integer;
begin
  select
    data_nascimento,
    saldo_pontos,
    saldo_banco_horas_minutos
  into
    v_data_nascimento,
    v_saldo_pontos,
    v_saldo_banco_horas
  from public.militares
  where id = new.militar_id;

  if not found then
    raise exception 'Militar nao encontrado.';
  end if;

  v_pontos_disponiveis := v_saldo_pontos;
  v_horas_disponiveis := v_saldo_banco_horas;

  if tg_op = 'UPDATE' and old.militar_id = new.militar_id then
    v_pontos_disponiveis := v_pontos_disponiveis + old.pontos_descontados;
    v_horas_disponiveis := v_horas_disponiveis + coalesce(old.horas_utilizadas_minutos, 0);
  end if;

  if new.motivo = 'produtividade' then
    new.pontos_descontados := 300;
    new.horas_utilizadas_minutos := null;

    if v_pontos_disponiveis < 300 then
      raise exception 'Militar nao possui 300 pontos disponiveis para folga por produtividade.';
    end if;
  elsif new.motivo = 'banco_horas' then
    new.pontos_descontados := 0;

    if coalesce(new.horas_utilizadas_minutos, 0) <= 0 then
      raise exception 'Informe horas_utilizadas_minutos para folga por banco de horas.';
    end if;

    if v_horas_disponiveis < new.horas_utilizadas_minutos then
      raise exception 'Militar nao possui saldo suficiente no banco de horas.';
    end if;
  elsif new.motivo = 'aniversario' then
    new.pontos_descontados := 0;
    new.horas_utilizadas_minutos := null;

    if to_char(new.data_folga, 'MM-DD') <> to_char(v_data_nascimento, 'MM-DD') then
      raise exception 'A folga por aniversario deve ser cadastrada na data do aniversario do militar.';
    end if;
  end if;

  return new;
end;
$$;

create or replace function public.fn_sincronizar_saldos_folga()
returns trigger
language plpgsql
as $$
begin
  if tg_op = 'INSERT' then
    update public.militares
       set saldo_pontos = saldo_pontos - new.pontos_descontados,
           saldo_banco_horas_minutos = saldo_banco_horas_minutos - coalesce(new.horas_utilizadas_minutos, 0)
     where id = new.militar_id;

    return new;
  end if;

  if tg_op = 'DELETE' then
    update public.militares
       set saldo_pontos = saldo_pontos + old.pontos_descontados,
           saldo_banco_horas_minutos = saldo_banco_horas_minutos + coalesce(old.horas_utilizadas_minutos, 0)
     where id = old.militar_id;

    return old;
  end if;

  if tg_op = 'UPDATE' then
    if old.militar_id = new.militar_id then
      update public.militares
         set saldo_pontos = saldo_pontos + old.pontos_descontados - new.pontos_descontados,
             saldo_banco_horas_minutos = saldo_banco_horas_minutos + coalesce(old.horas_utilizadas_minutos, 0) - coalesce(new.horas_utilizadas_minutos, 0)
       where id = new.militar_id;
    else
      update public.militares
         set saldo_pontos = saldo_pontos + old.pontos_descontados,
             saldo_banco_horas_minutos = saldo_banco_horas_minutos + coalesce(old.horas_utilizadas_minutos, 0)
       where id = old.militar_id;

      update public.militares
         set saldo_pontos = saldo_pontos - new.pontos_descontados,
             saldo_banco_horas_minutos = saldo_banco_horas_minutos - coalesce(new.horas_utilizadas_minutos, 0)
       where id = new.militar_id;
    end if;

    return new;
  end if;

  return null;
end;
$$;

drop trigger if exists trg_militares_set_updated_at on public.militares;
create trigger trg_militares_set_updated_at
before update on public.militares
for each row
execute function public.fn_set_updated_at();

drop trigger if exists trg_registros_produtividade_set_updated_at on public.registros_produtividade;
create trigger trg_registros_produtividade_set_updated_at
before update on public.registros_produtividade
for each row
execute function public.fn_set_updated_at();

drop trigger if exists trg_folgas_set_updated_at on public.folgas;
create trigger trg_folgas_set_updated_at
before update on public.folgas
for each row
execute function public.fn_set_updated_at();

drop trigger if exists trg_preparar_registro_produtividade on public.registros_produtividade;
create trigger trg_preparar_registro_produtividade
before insert or update on public.registros_produtividade
for each row
execute function public.fn_preparar_registro_produtividade();

drop trigger if exists trg_sincronizar_pontos_produtividade on public.registros_produtividade;
create trigger trg_sincronizar_pontos_produtividade
after insert or update or delete on public.registros_produtividade
for each row
execute function public.fn_sincronizar_pontos_produtividade();

drop trigger if exists trg_preparar_folga on public.folgas;
create trigger trg_preparar_folga
before insert or update on public.folgas
for each row
execute function public.fn_preparar_folga();

drop trigger if exists trg_sincronizar_saldos_folga on public.folgas;
create trigger trg_sincronizar_saldos_folga
after insert or update or delete on public.folgas
for each row
execute function public.fn_sincronizar_saldos_folga();

create or replace view public.vw_militares_com_direito_folga_produtividade as
select
  m.id,
  m.matricula,
  m.cpf,
  m.nome_completo,
  m.nome_guerra,
  m.graduacao,
  m.saldo_pontos,
  (m.saldo_pontos / 300) as quantidade_folgas_disponiveis,
  m.saldo_banco_horas_minutos
from public.militares m
where m.ativo = true
  and m.saldo_pontos >= 300;

create or replace view public.vw_produtividade_anual_militares as
select
  m.id as militar_id,
  m.matricula,
  m.nome_completo,
  m.nome_guerra,
  extract(year from rp.data_hora_ocorrencia at time zone 'America/Sao_Paulo')::int as ano,
  count(*) as quantidade_registros,
  sum(rp.pontos_creditados) as pontos_produzidos,
  max(rp.data_hora_ocorrencia) as ultima_ocorrencia
from public.militares m
join public.registros_produtividade rp
  on rp.militar_id = m.id
group by
  m.id,
  m.matricula,
  m.nome_completo,
  m.nome_guerra,
  extract(year from rp.data_hora_ocorrencia at time zone 'America/Sao_Paulo');
```

## 3. O que este script entrega

### Tabela `public.militares`

Guarda os dados cadastrais do policial militar:

1. Matricula
2. CPF
3. Nome completo
4. Nome de guerra
5. Graduacao
6. Data de nascimento
7. Saldo de pontos
8. Saldo de banco de horas

### Tabela `public.registros_produtividade`

Registra a produtividade por militar, com:

1. Numero da ocorrencia
2. Militar vinculado
3. Nome de guerra no momento do registro
4. Tipo de ocorrencia
5. Quantidade de drogas em gramas, quando aplicavel
6. Pontos calculados automaticamente
7. Detalhes
8. Data e hora da ocorrencia

### Tabela `public.folgas`

Registra a concessao de folga, com:

1. Militar
2. Data da folga
3. Motivo: `banco_horas`, `produtividade` ou `aniversario`
4. Status: `pendente` ou `concluida`
5. Horas utilizadas em minutos, quando a folga for por banco de horas
6. Pontos descontados automaticamente, quando a folga for por produtividade
7. Observacao

### Views

`public.vw_militares_com_direito_folga_produtividade`

1. Mostra os militares ativos com `saldo_pontos >= 300`
2. Informa quantas folgas ja podem ser concedidas com base na divisao por 300

`public.vw_produtividade_anual_militares`

1. Resume a produtividade anual
2. Ajuda na analise de desempenho ao longo do ano

## 4. Regras de pontuacao implementadas

1. `arma_de_fogo`: 300 pontos
2. `drogas` de 0 a 499 gramas: 75 pontos
3. `drogas` de 500 a 999 gramas: 150 pontos
4. `drogas` acima de 999 gramas: 300 pontos
5. `TCO`: 75 pontos
6. `prisao_homicida`: 75 pontos
7. `apreensao_veiculo`: 75 pontos

## 5. Consultas uteis

### Listar militares com direito a folga por produtividade

```sql
select *
from public.vw_militares_com_direito_folga_produtividade
order by saldo_pontos desc, nome_guerra asc;
```

### Consultar produtividade anual

```sql
select *
from public.vw_produtividade_anual_militares
where ano = 2026
order by pontos_produzidos desc, nome_guerra asc;
```

### Exemplo de cadastro de militar

```sql
insert into public.militares (
  matricula,
  cpf,
  nome_completo,
  nome_guerra,
  graduacao,
  data_nascimento,
  saldo_banco_horas_minutos
)
values (
  '123456',
  '12345678901',
  'Joao da Silva',
  'Silva',
  'cabo',
  '1990-04-15',
  480
);
```

### Exemplo de registro de produtividade por arma de fogo

```sql
insert into public.registros_produtividade (
  numero_ocorrencia,
  militar_id,
  tipo_ocorrencia,
  detalhes,
  data_hora_ocorrencia
)
values (
  '2026-000123',
  'SUBSTITUA_PELO_UUID_DO_MILITAR',
  'arma_de_fogo',
  'Apreensao de arma de fogo em abordagem.',
  now()
);
```

### Exemplo de registro de produtividade por drogas

```sql
insert into public.registros_produtividade (
  numero_ocorrencia,
  militar_id,
  tipo_ocorrencia,
  quantidade_drogas_gramas,
  detalhes,
  data_hora_ocorrencia
)
values (
  '2026-000124',
  'SUBSTITUA_PELO_UUID_DO_MILITAR',
  'drogas',
  650,
  'Apreensao de entorpecentes.',
  now()
);
```

### Exemplo de cadastro de folga por produtividade

```sql
insert into public.folgas (
  militar_id,
  data_folga,
  motivo,
  status,
  observacao
)
values (
  'SUBSTITUA_PELO_UUID_DO_MILITAR',
  '2026-04-20',
  'produtividade',
  'pendente',
  'Folga concedida apos atingir 300 pontos.'
);
```

### Exemplo de cadastro de folga por banco de horas

```sql
insert into public.folgas (
  militar_id,
  data_folga,
  motivo,
  status,
  horas_utilizadas_minutos,
  observacao
)
values (
  'SUBSTITUA_PELO_UUID_DO_MILITAR',
  '2026-04-22',
  'banco_horas',
  'pendente',
  480,
  'Folga concedida com uso de 8 horas do banco.'
);
```

### Exemplo de cadastro de folga por aniversario

```sql
insert into public.folgas (
  militar_id,
  data_folga,
  motivo,
  status,
  observacao
)
values (
  'SUBSTITUA_PELO_UUID_DO_MILITAR',
  '2026-04-15',
  'aniversario',
  'pendente',
  'Folga de aniversario.'
);
```

## 6. Melhorias recomendadas para a proxima etapa

Se quiser evoluir este modelo depois, as proximas melhorias naturais sao:

1. Criar politicas RLS do Supabase para limitar acesso por perfil.
2. Criar uma tabela de usuarios/perfis para separar administrador e operador.
3. Criar uma tabela de historico de banco de horas para auditar creditos e debitos.
4. Criar uma tabela de auditoria para rastrear alteracoes em produtividade e folgas.
5. Criar procedures para concessao automatica de folga quando o militar atingir determinado saldo.
