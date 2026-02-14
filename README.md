# Onboarding DBA: Guia de Ambiente e Ferramentas

Jhenifer, bem-vinda ao time de Dados da **RH Labs**! Este documento serve como um mapa para você se localizar nos nossos sistemas, ferramentas e processos no dia a dia.

## Stack de Ferramentas

Categoria | Ferramenta |  Descrição/ Uso
--------- | ---------- | --------------
**SGBDs**     | SQLite e Mysql | É o nosso motor de gerenciamento do Banco de dados. Você se pergunta: Por que dois? Atualmente estamos usando somente o SQLite, pois é um MVP e para aumentar a velocidade do gerenciamento do banco, inicialmente optamos por usar o SQLite, mas, futuramente, conforme o negócio escala, planejaremos a migração do banco para o Mysql.
**IDE DB**    | phpMyAdmin (ou PG Admin) | Eu recomendo que use a ferramenta que seja mais confortável para ti, mas o PHP já trás a ferramenta pgAdmin nativa. Se quiser, pode dar uma olhada e testar para ver se você se adapta. 
**IDE Code** | VSCode ou o Storm |  Atualmente eu uso o vscode, mas você pode usar o Inletij Storm por 1 ano de graça por ser estudante. as Extensões do vscode que eu uso nesse projeto são: Container Tools (Microsoft), Laravel (Laravel), PHP (DEVSENSE), PostgreSQL(Microsoft), SQLite (alexcvzz), SQLite Viewer (Florian Klampfer).
**Ambiente Dev** | Docker / Laravel Sail | Eu, particularmente, não gosto de ficar instalando coisas no meu linux porque de 3 em 3 meses eu estou testando uma distro nova. Por conta disso, para evitar todo o retrabalho de instalar 4 linguagens, 10 ferramentas, um monte de coisas que vão ser apagadas em 3 meses, eu uso o Docker. O **Docker** é seu amigo no final de tudo. Ele é tipo o amigo chato de viajar, porque antes da viagem fica querendo confirmar tudo, passagem, dinheiro pro hotel, planeja quanto vai gastar por dia e quer dizer até quantas meias você vai levar na viagem, contudo, no dia da viagem, você só vai precisar mandar um `$ docker compose up -d` e tudo funcionará perfeitamente. O "**Sail**" é uma ferramenta própria do laravel que ajuda a orquestrar todos os itens usados no Laravel, por exemplo, você dá um ```bash $ composer require lravel/sail && php artisan sail:install && ./vendor/bin/sail up ``` e você já tem o laravel conectado e integrado com redis, typesense, valkey, meilisearch e etc
**Acesso Remoto** | SSH | Nós teremos acesso ao banco de dados de produção diretamente. Fique tranquila que nenhuma grande responsabilidade sobrecairá sobre você neste sentido, porém, somente eu, tu e jeff teremos as chaves do portão de Mordor.
**Documentação** | Mattermost | Iremos, no decorrer do desenvolvimento, agregar toda documentação dentro do mattermost da RH Labs, para deixar ela pública e acessível para todos. É importante ressaltar que a tarefa só se dar por concluída quando envia junto da documentação. Então, mesmo que demore 3 dias pra fazer a tarefa e mais 1 dia para fazer a documentação, é melhor qque leve esses 4 dias para fechar o fluxo da tarefa de modo completo. Assim manteremos tudo documentado e bem organizado.

## Topologia e Acessos
1. Ambientes
    1.  Produção (prod): Onde o Tio Ben fala "Com grandes poderes, vem grandes responsabilidades". Você terá acesso à prod, até porque eu quero que você saia com experiência de mexer, quebrar e consertar ambientes em produção. Se tudo der certo, você vai surtar pelo menos 1 vez tendo que descobrir o que qubrou no banco, mas fica tranquila que estaremos juntos nesse caminho.
    2.  Desenvolvimento (dev): Aqui você tem liberdade para testar e quebrar as queries. Lembre-se de nunca subir no git o `*.lock` e não instalar dependências sem que sejam discutidas antes. Eu recomendo sempre usar o docker, porque o docker inibe o erro de "na minha máquina funciona" já que o docker nivela o jogo para todas as máquinas.
3. Primeiro Passo: Credenciais
  1.   IMPORTANT: Nunca compartilhe senhas por canais não seguros. 

## Processos do Dia a Dia
### Check-list antes de codar
1. Verificar se os Backups do servidor rodaram com sucesso
2. Checar alertas de espaço em disco no servidor
3. Olhar o painel de Slow Queries (No pgAdmin, fica em **Status** -> **Monitor**
#### Integração com Desenvolvimento (Laravel)
Como nós usamos o Laravel no projeto, a nossa interação com o banco de dados n~]ao é apenas via SQL puro, mas sim através de abstrações que garantem a integridade e a padronização do código.

<details>
<summary><b>[IMAGEM] Arquitetura de Software não oficial (O doc oficial é o feito pelo Luan)</b></summary>
    <img alt="Arquitetura" src="https://github.com/xtirian/onboarding-dba-jhenifer/blob/main/Arquitetura%20de%20Software%20(RHLabs).png?raw=true" />
</details>

1. Migrations: Fazem o controle de versão do banco
  1. Nunca crie tabelas manualmente via console/IDE de produção
  2. Toda alteração de schema deve ser feita via `$ php artisan make:migration` ou `$ sail artisan migrate`
  3. "Fiz uma coisa errada e quero dar Rollback". Sempre teste o método down() da sua migration para garantir que a alteração pode ser desfeita.

Exemplo:

**a. Você primeiro cria a Migration**
```bash
    $ php (ou sail) artisan make:migration create_table_leads
```
**b. Vicê define a estrutura da tabela. Quando você abrir a pasta `database/migrations/*.php` ela estará vazia só com a estrutura básica do `public function up(): void { ... }`**
```php
// Isto aqui dentro da migration

public function up(): void
{
    Schema::create('leads', function (Blueprint $table) {
        // Se você fez o que te indiquei para dar uma olhada em POO com PHP, isso aqui é moleza para entender:

        $table->uuid('id')->primary(); // Aqui você marca que o ID é UUID
        $table->uuid('session_id')->nullable()->index(); // Aqui você cria o índice
        $table->string('name', 100)->nullable(); // Aqui define o tipo String e tamanho 100
        $table->string('email', 100)->nullable()->unique(); 
        $table->string('phone', 20)->nullable();
        
        $table->enum('status', ['draft', 'converted'])->default('draft');
        
        $table->string('last_field_filled', 50)->nullable();
        $table->timestamp('converted_at')->nullable();
        $table->timestamps();
    });
}
```
**c. Por fim você executa a migrate**
```bash
    $php (ou sail) arisan migrate
```

2. Models (Eloquent)
   Os models são a presentação das suas tabelas no código PHP
  1. **Nomeclatura** Tableas no plural (`users`), Models no singular (`User`);
  2. **Fillable** Sempre defina a propriedade `$fillable` para evitar ataques de *Mass Assignment*. Isso diminui o esforço do servidor na hora de responder uma solicitação, porque define na base o que será entregue ao solicitante sobre aquele model [Vídeo de um indiano sobre o assunto! High Tier +8000](https://www.youtube.com/watch?v=epoFy-co81U).
  3. **Relacionamentos** Defina explicitament os métodos `hasMany`, `belongsTo`, etc., para facilitar o uso de *Eager Loading*  e evitar o problema de N+1 queries. [Lazy Loading vs. Eager Loading in Laravel](https://medium.com/@mhaseeb_01/lazy-loading-vs-eager-loading-in-laravel-3d5192b75be9)


Exemplo:


**a. Criar o model é bem simples**
```bash
$ php (ou sail) artisan make:model Lead
```

**b. Depois você poderá editar o arquivo de Model**
```php
<?php
namespace App\Models;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Factories\HasFactory;

class Lead extends Model
{
    use HasFactory, HasUuids; 

    // Configurações de Chave Primária (UUID)
    protected $keyType = 'string';
    public $incrementing = false;

    // Sempre definir a propriedade $fillable e para entender melhor, veja o video enviado antes
    protected $fillable = [
        'session_id',
        'name',
        'email',
        'phone',
        'status',
        'last_field_filled',
        'converted_at',
    ];

    /**
     * Casts de Atributos
     * Garante que o 'converted_at' seja tratado como um objeto Carbon (Data)
     * Jhen, se quiser saber mais sobre isto, ver: https://laravel.com/docs/12.x/eloquent-mutators#date-casting
     */
    protected $casts = [
        'converted_at' => 'datetime',
        'created_at'   => 'datetime',
        'updated_at'   => 'datetime',
    ];

    /**
     * Scope para filtrar apenas leads convertidos
     * Facilita o reuso de queries no Repository
     */
    public function scopeConverted($query)
    {
        return $query->where('status', 'converted');
    }
}


```
5. Repository Pattern
  Utyilizamos Repositories para isolar a lógica de acesso aos dados da lógica de negócio (Services e Controllers).
  * **Objetivo** é que se precisarmos mudar a query, ou método de busca, não precisaremo mudar em todos os lugares que esta query é usada, mudamos apenas o repository
  * A **Dica** é que o Repository deve retornar collections ou models, mantendo o Controller/Service só com o essencial.

Exemploooo:
**a. Aqui você precisa criar diretamente o Repository.** Fica o entendimento que o uso de Repositories é uma abstração do fluxo de um Software, ou se você preferir, ele é um _Pattern_ muito recomendado e eu gostaria muito que você desse uma olhada neste artigo: https://medium.com/by-vinicius-reis/repository-pattern-n%C3%A3o-precisa-ser-chato-principalmente-com-laravel-d97235b31c7e


```php
<?php
namespace App\Repositories;
use App\Models\Lead;
use Illuminate\Support\Collection;
use Carbon\Carbon;

class LeadRepository
{
    protected $model;
    public function __construct(Lead $lead)
    {
        $this->model = $lead;
    }    
    /**
     * REGRAS DE NEGÓCIO ABSTRAÍDAS:
     * O Controller não precisa saber que existe um campo 'converted_at'
     * ou que o status é uma string 'converted'. Ele só pede para 'converter'.
     */
    public function markAsConverted(string $id): bool
    {
        $lead = $this->model->findOrFail($id);
        
        /**
         * Em alguns casos, as regras de negócio mais simples podem ser executada dentro do Repository e o service pode se preocupar com integrações de APIs, conexões entre serviços e regras mais pesadas. Para este caso aqui, o dev que está construindo o serviço poderia focar em regras de negócio enquanto o repository se preocupe com o quê de fato será atualizado.
         * O dev pode chamar somente:
            public function convertLead(string $id)
            {
                $lead = $this->repository->find($id);
                // REGRA DE NEGÓCIO: Aqui é o lugar dela!
                if (empty($lead->email)) {
                    throw new \Exception("Lead sem e-mail não pode ser convertido.");
                }
            
                return $this->repository->markAsConverted($id); <- Olha aqui o que precisa chamar
            }
         */
        return $lead->update([
            'status'       => 'converted',
            'converted_at' => now(), // Centralizado aqui, evita que o dev esqueça a data
        ]);
    }
    /**
     * IMPORTÂNCIA: Padronização
     * Se mudarmos a regra de "leads recentes" de 10 para 20 dias, 
     * ou se adicionarmos um filtro para ignorar leads de teste (@test.com),
     * alteramos apenas aqui e reflete em todo o sistema.
     */
    public function getOnlyConvertedLeads()
    {
        return $this->model
            ->whereNotNull('email') 
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->get();
    }

    /**
     * Retorna os leads mais recentes para o Dashboard do DBA;
     */
    public function getLatestLeads(int $limit = 10): Collection
    {
        return $this->model->orderBy('created_at', 'desc')->limit($limit)->get();
    }
}
```

## Dicas
* Na dúvida, pergunte.
* Se for fazer qualquer alteração no Banco, seja local ou produção. Faça um dump (backup) do banco `$  mysqldump -u [usuario] -p [nome_do_banco] > [backup-YYMMDDHHMM].sql` (YY->Ano, MM->Mês, DD->Dia, HH->Hora, MM->Minuto) **OU** `$ sqlite3 [nome_do_banco].db .dump > [backup-YYMMDDHHMM].sql`
