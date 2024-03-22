# 🚚 Sistema de Bilhetagem para Logística de Caixas

Sistema SaaS de bilhetagem virtual para transporte e logística de caixas, onde cada bilhete corresponde a um deslocamento medido em peso e tamanho. A origem emite o bilhete e o destino valida, calculando automaticamente os valores para a transportadora e manutenção do SaaS.

## 🛠️ Stack Tecnológica

- **Backend**: PHP 8+ com CodeIgniter 4 (API REST)
- **Frontend**: ReactJS 18+
- **Banco de Dados**: MySQL 8.0
- **Containerização**: Docker & Docker Compose

---

## 📁 Estrutura do Projeto

```
logistics-ticketing/
├── backend/
│   ├── app/
│   │   ├── Config/
│   │   │   ├── Routes.php
│   │   │   ├── Database.php
│   │   │   ├── Filters.php
│   │   │   └── Cors.php
│   │   ├── Controllers/
│   │   │   └── API/
│   │   │       ├── AuthController.php
│   │   │       ├── TicketController.php
│   │   │       ├── CompanyController.php
│   │   │       ├── ValidationController.php
│   │   │       └── BillingController.php
│   │   ├── Models/
│   │   │   ├── CompanyModel.php
│   │   │   ├── UserModel.php
│   │   │   ├── TicketModel.php
│   │   │   ├── ValidationModel.php
│   │   │   ├── PricingModel.php
│   │   │   └── TransactionModel.php
│   │   ├── Filters/
│   │   │   └── AuthFilter.php
│   │   └── Libraries/
│   │       ├── JWTHandler.php
│   │       └── PricingCalculator.php
│   ├── public/
│   │   └── index.php
│   ├── writable/
│   ├── .env
│   ├── composer.json
│   └── Dockerfile
├── frontend/
│   ├── public/
│   │   └── index.html
│   ├── src/
│   │   ├── components/
│   │   │   ├── Auth/
│   │   │   │   ├── Login.jsx
│   │   │   │   └── Register.jsx
│   │   │   ├── Tickets/
│   │   │   │   ├── CreateTicket.jsx
│   │   │   │   ├── TicketList.jsx
│   │   │   │   └── TicketDetail.jsx
│   │   │   ├── Validation/
│   │   │   │   └── ValidateTicket.jsx
│   │   │   ├── Dashboard/
│   │   │   │   └── Dashboard.jsx
│   │   │   └── Billing/
│   │   │       ├── Transactions.jsx
│   │   │       └── Reports.jsx
│   │   ├── services/
│   │   │   └── api.js
│   │   ├── contexts/
│   │   │   └── AuthContext.js
│   │   ├── utils/
│   │   │   └── helpers.js
│   │   ├── App.js
│   │   └── index.js
│   ├── package.json
│   ├── .env
│   └── Dockerfile
├── database/
│   └── init.sql
├── docker-compose.yml
└── README.md
```

---

## 🗄️ Banco de Dados

### Arquivo: `database/init.sql`

```sql
-- ============================================
-- SISTEMA DE BILHETAGEM LOGÍSTICA
-- Database Schema
-- ============================================

SET SQL_MODE = "NO_AUTO_VALUE_ON_ZERO";
SET time_zone = "+00:00";

-- ============================================
-- TABELA: companies
-- ============================================
CREATE TABLE companies (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    cnpj VARCHAR(18) UNIQUE NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    address TEXT,
    type ENUM('origin', 'destination', 'both') DEFAULT 'both' COMMENT 'Tipo de operação da empresa',
    api_key VARCHAR(255) UNIQUE COMMENT 'Chave API para integração',
    status ENUM('active', 'inactive', 'suspended') DEFAULT 'active',
    monthly_fee DECIMAL(10,2) DEFAULT 0.00 COMMENT 'Taxa mensal SaaS',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_cnpj (cnpj),
    INDEX idx_api_key (api_key),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================
-- TABELA: users
-- ============================================
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    company_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    password VARCHAR(255) NOT NULL,
    role ENUM('admin', 'operator', 'validator') DEFAULT 'operator' COMMENT 'admin: gerencia tudo | operator: emite bilhetes | validator: valida entregas',
    status ENUM('active', 'inactive') DEFAULT 'active',
    last_login TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (company_id) REFERENCES companies(id) ON DELETE CASCADE,
    UNIQUE KEY unique_email (email),
    INDEX idx_company (company_id),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================
-- TABELA: tickets
-- ============================================
CREATE TABLE tickets (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ticket_code VARCHAR(100) UNIQUE NOT NULL COMMENT 'Código único do bilhete',
    qr_code TEXT COMMENT 'QR Code em base64 ou URL',
    
    -- ORIGEM
    origin_company_id INT NOT NULL,
    origin_user_id INT NOT NULL COMMENT 'Usuário que emitiu',
    origin_address TEXT,
    
    -- DESTINO
    destination_company_id INT NOT NULL,
    destination_address TEXT,
    destination_contact VARCHAR(255),
    
    -- INFORMAÇÕES DA CARGA
    box_count INT NOT NULL DEFAULT 1 COMMENT 'Quantidade de caixas',
    total_weight DECIMAL(10,3) NOT NULL COMMENT 'Peso total em KG',
    total_volume DECIMAL(10,4) NOT NULL COMMENT 'Volume total em m³',
    
    -- DIMENSÕES DETALHADAS (JSON)
    dimensions JSON COMMENT 'Array de objetos: [{width, height, depth, weight, description}]',
    
    -- DESCRIÇÃO DA CARGA
    cargo_description TEXT,
    special_instructions TEXT,
    
    -- STATUS DO BILHETE
    status ENUM('pending', 'in_transit', 'validated', 'cancelled') DEFAULT 'pending',
    
    -- VALORES CALCULADOS
    transport_value DECIMAL(10,2) DEFAULT 0.00 COMMENT 'Valor do transporte',
    saas_fee DECIMAL(10,2) DEFAULT 0.00 COMMENT 'Taxa SaaS',
    total_value DECIMAL(10,2) DEFAULT 0.00 COMMENT 'Valor total',
    
    -- CONTROLE DE DATAS
    issued_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    estimated_delivery TIMESTAMP NULL,
    validated_at TIMESTAMP NULL,
    cancelled_at TIMESTAMP NULL,
    cancellation_reason TEXT,
    
    -- METADADOS ADICIONAIS
    metadata JSON COMMENT 'Dados extras: tracking, notas, etc',
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (origin_company_id) REFERENCES companies(id),
    FOREIGN KEY (destination_company_id) REFERENCES companies(id),
    FOREIGN KEY (origin_user_id) REFERENCES users(id),
    
    INDEX idx_ticket_code (ticket_code),
    INDEX idx_status (status),
    INDEX idx_origin (origin_company_id),
    INDEX idx_destination (destination_company_id),
    INDEX idx_issued_at (issued_at),
    INDEX idx_dates (issued_at, validated_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================
-- TABELA: validations
-- ============================================
CREATE TABLE validations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ticket_id INT NOT NULL,
    validator_user_id INT NOT NULL,
    
    -- DADOS CONFERIDOS NA ENTREGA
    validated_weight DECIMAL(10,3) COMMENT 'Peso real conferido (kg)',
    validated_volume DECIMAL(10,4) COMMENT 'Volume real conferido (m³)',
    validated_box_count INT COMMENT 'Quantidade real de caixas',
    
    -- CÁLCULO DE DIFERENÇAS
    weight_difference DECIMAL(10,3) COMMENT 'Diferença de peso (positivo = maior | negativo = menor)',
    volume_difference DECIMAL(10,4) COMMENT 'Diferença de volume',
    box_count_difference INT COMMENT 'Diferença na quantidade',
    
    -- EVIDÊNCIAS
    notes TEXT COMMENT 'Observações do validador',
    photos JSON COMMENT 'URLs das fotos tiradas na validação',
    signature TEXT COMMENT 'Assinatura digital ou imagem',
    
    -- LOCALIZAÇÃO DA VALIDAÇÃO
    latitude DECIMAL(10,8),
    longitude DECIMAL(11,8),
    
    -- CONTROLE DE QUALIDADE
    damage_report TEXT COMMENT 'Relato de avarias',
    conformity ENUM('total', 'partial', 'non_conforming') DEFAULT 'total',
    
    validated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (ticket_id) REFERENCES tickets(id) ON DELETE CASCADE,
    FOREIGN KEY (validator_user_id) REFERENCES users(id),
    
    INDEX idx_ticket (ticket_id),
    INDEX idx_validator (validator_user_id),
    INDEX idx_validated_at (validated_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================
-- TABELA: pricing_rules
-- ============================================
CREATE TABLE pricing_rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    
    -- VALORES BASE
    base_price DECIMAL(10,2) DEFAULT 0.00 COMMENT 'Preço base por operação',
    price_per_kg DECIMAL(10,2) DEFAULT 0.00 COMMENT 'Valor por kg',
    price_per_m3 DECIMAL(10,2) DEFAULT 0.00 COMMENT 'Valor por m³',
    price_per_box DECIMAL(10,2) DEFAULT 0.00 COMMENT 'Valor por caixa',
    
    -- TAXA SAAS
    saas_percentage DECIMAL(5,2) DEFAULT 5.00 COMMENT 'Porcentagem da taxa SaaS',
    saas_fixed_fee DECIMAL(10,2) DEFAULT 0.00 COMMENT 'Taxa fixa SaaS adicional',
    
    -- DESCONTOS POR VOLUME (JSON)
    weight_tiers JSON COMMENT '[{min: 0, max: 100, discount_percentage: 0}, {...}]',
    volume_tiers JSON COMMENT '[{min: 0, max: 5, discount_percentage: 0}, {...}]',
    
    -- ACRÉSCIMOS
    fragile_cargo_fee DECIMAL(10,2) DEFAULT 0.00 COMMENT 'Taxa para carga frágil',
    express_delivery_fee DECIMAL(10,2) DEFAULT 0.00 COMMENT 'Taxa para entrega expressa',
    
    -- VIGÊNCIA DA REGRA
    valid_from DATE,
    valid_until DATE,
    
    is_active BOOLEAN DEFAULT TRUE,
    is_default BOOLEAN DEFAULT FALSE,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_active (is_active),
    INDEX idx_dates (valid_from, valid_until)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================
-- TABELA: transactions
-- ============================================
CREATE TABLE transactions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ticket_id INT NOT NULL,
    
    -- VALORES DETALHADOS
    transport_amount DECIMAL(10,2) NOT NULL COMMENT 'Valor do frete',
    saas_amount DECIMAL(10,2) NOT NULL COMMENT 'Taxa SaaS',
    discount_amount DECIMAL(10,2) DEFAULT 0.00 COMMENT 'Descontos aplicados',
    additional_fees DECIMAL(10,2) DEFAULT 0.00 COMMENT 'Taxas adicionais',
    total_amount DECIMAL(10,2) NOT NULL COMMENT 'Valor total',
    
    -- RESPONSÁVEL PELO PAGAMENTO
    payer_company_id INT NOT NULL COMMENT 'Empresa pagadora',
    
    -- CONTROLE DE PAGAMENTO
    status ENUM('pending', 'paid', 'cancelled', 'refunded', 'overdue') DEFAULT 'pending',
    payment_method VARCHAR(50) COMMENT 'Forma de pagamento',
    payment_reference VARCHAR(255) COMMENT 'Referência/ID do pagamento externo',
    payment_gateway VARCHAR(100) COMMENT 'Gateway utilizado',
    
    -- DATAS
    due_date DATE COMMENT 'Data de vencimento',
    paid_at TIMESTAMP NULL,
    refunded_at TIMESTAMP NULL,
    
    -- OBSERVAÇÕES
    notes TEXT,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (ticket_id) REFERENCES tickets(id),
    FOREIGN KEY (payer_company_id) REFERENCES companies(id),
    
    INDEX idx_ticket (ticket_id),
    INDEX idx_payer (payer_company_id),
    INDEX idx_status (status),
    INDEX idx_dates (created_at, paid_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================
-- TABELA: audit_logs
-- ============================================
CREATE TABLE audit_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    company_id INT,
    
    entity_type VARCHAR(50) COMMENT 'ticket, validation, transaction, company, user',
    entity_id INT COMMENT 'ID da entidade afetada',
    action VARCHAR(50) COMMENT 'create, update, delete, validate, cancel',
    
    old_values JSON COMMENT 'Valores antes da alteração',
    new_values JSON COMMENT 'Valores depois da alteração',
    
    ip_address VARCHAR(45),
    user_agent TEXT,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (company_id) REFERENCES companies(id) ON DELETE SET NULL,
    
    INDEX idx_entity (entity_type, entity_id),
    INDEX idx_user (user_id),
    INDEX idx_company (company_id),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================
-- DADOS INICIAIS
-- ============================================

-- Inserir regra de precificação padrão
INSERT INTO pricing_rules (
    name, 
    description,
    base_price, 
    price_per_kg, 
    price_per_m3, 
    price_per_box, 
    saas_percentage,
    saas_fixed_fee,
    valid_from, 
    is_active,
    is_default
) VALUES (
    'Tabela Padrão 2025',
    'Regra de precificação padrão do sistema',
    10.00,  -- R$ 10,00 de base
    2.50,   -- R$ 2,50 por kg
    15.00,  -- R$ 15,00 por m³
    5.00,   -- R$ 5,00 por caixa
    5.00,   -- 5% de taxa SaaS
    2.00,   -- R$ 2,00 fixo SaaS
    '2025-01-01',
    TRUE,
    TRUE
);

-- Exemplo de faixas de desconto (pode editar depois)
UPDATE pricing_rules 
SET weight_tiers = JSON_ARRAY(
    JSON_OBJECT('min', 0, 'max', 50, 'discount_percentage', 0),
    JSON_OBJECT('min', 51, 'max', 200, 'discount_percentage', 5),
    JSON_OBJECT('min', 201, 'max', 500, 'discount_percentage', 10),
    JSON_OBJECT('min', 501, 'max', 999999, 'discount_percentage', 15)
),
volume_tiers = JSON_ARRAY(
    JSON_OBJECT('min', 0, 'max', 1, 'discount_percentage', 0),
    JSON_OBJECT('min', 1.01, 'max', 5, 'discount_percentage', 5),
    JSON_OBJECT('min', 5.01, 'max', 10, 'discount_percentage', 10),
    JSON_OBJECT('min', 10.01, 'max', 999999, 'discount_percentage', 15)
)
WHERE id = 1;

-- ============================================
-- VIEWS ÚTEIS
-- ============================================

-- View: Resumo de Tickets por Empresa
CREATE VIEW v_tickets_summary AS
SELECT 
    c.id as company_id,
    c.name as company_name,
    COUNT(CASE WHEN t.status = 'pending' THEN 1 END) as pending_tickets,
    COUNT(CASE WHEN t.status = 'in_transit' THEN 1 END) as in_transit_tickets,
    COUNT(CASE WHEN t.status = 'validated' THEN 1 END) as validated_tickets,
    COUNT(CASE WHEN t.status = 'cancelled' THEN 1 END) as cancelled_tickets,
    COUNT(t.id) as total_tickets,
    SUM(t.total_value) as total_revenue
FROM companies c
LEFT JOIN tickets t ON (c.id = t.origin_company_id OR c.id = t.destination_company_id)
GROUP BY c.id, c.name;

-- View: Transações Pendentes
CREATE VIEW v_pending_transactions AS
SELECT 
    tr.id,
    tr.ticket_id,
    t.ticket_code,
    c.name as payer_company,
    tr.total_amount,
    tr.due_date,
    DATEDIFF(CURDATE(), tr.due_date) as days_overdue,
    tr.created_at
FROM transactions tr
JOIN tickets t ON tr.ticket_id = t.id
JOIN companies c ON tr.payer_company_id = c.id
WHERE tr.status = 'pending'
ORDER BY tr.due_date ASC;
```

---

## 🔧 Backend - CodeIgniter 4

### 1. Configuração de Rotas

**Arquivo: `backend/app/Config/Routes.php`**

```php
<?php

namespace Config;

use CodeIgniter\Config\BaseConfig;

$routes = Services::routes();

$routes->setDefaultNamespace('App\Controllers');
$routes->setDefaultController('Home');
$routes->setDefaultMethod('index');
$routes->setTranslateURIDashes(false);
$routes->set404Override();
$routes->setAutoRoute(false);

// ============================================
// ROTAS DE AUTENTICAÇÃO
// ============================================
$routes->group('api/auth', function($routes) {
    $routes->post('login', 'API\AuthController::login');
    $routes->post('register', 'API\AuthController::register');
    $routes->get('me', 'API\AuthController::me', ['filter' => 'auth']);
    $routes->post('logout', 'API\AuthController::logout', ['filter' => 'auth']);
    $routes->post('refresh', 'API\AuthController::refresh');
});

// ============================================
// ROTAS DE EMPRESAS
// ============================================
$routes->group('api/companies', ['filter' => 'auth'], function($routes) {
    $routes->get('/', 'API\CompanyController::index');
    $routes->get('(:num)', 'API\CompanyController::show/$1');
    $routes->post('/', 'API\CompanyController::create');
    $routes->put('(:num)', 'API\CompanyController::update/$1');
    $routes->delete('(:num)', 'API\CompanyController::delete/$1');
    $routes->get('search', 'API\CompanyController::search');
});

// ============================================
// ROTAS DE BILHETES
// ============================================
$routes->group('api/tickets', ['filter' => 'auth'], function($routes) {
    $routes->get('/', 'API\TicketController::index');
    $routes->get('(:num)', 'API\TicketController::show/$1');
    $routes->post('/', 'API\TicketController::create');
    $routes->put('(:num)', 'API\TicketController::update/$1');
    $routes->delete('(:num)', 'API\TicketController::cancel/$1');
    
    // Busca por código
    $routes->get('code/(:segment)', 'API\TicketController::getByCode/$1');
    
    // Estatísticas
    $routes->get('stats/overview', 'API\TicketController::stats');
});

// ============================================
// ROTAS DE VALIDAÇÃO
// ============================================
$routes->group('api/validations', ['filter' => 'auth'], function($routes) {
    $routes->post('/', 'API\ValidationController::validate');
    $routes->get('ticket/(:num)', 'API\ValidationController::getByTicket/$1');
    $routes->get('history', 'API\ValidationController::history');
});

// ============================================
// ROTAS DE FATURAMENTO
// ============================================
$routes->group('api/billing', ['filter' => 'auth'], function($routes) {
    $routes->get('dashboard', 'API\BillingController::dashboard');
    $routes->get('transactions', 'API\BillingController::transactions');
    $routes->get('report', 'API\BillingController::report');
    $routes->post('pay/(:num)', 'API\BillingController::markAsPaid/$1');
});

// ============================================
// ROTAS DE PRECIFICAÇÃO (Admin)
// ============================================
$routes->group('api/pricing', ['filter' => 'auth'], function($routes) {
    $routes->get('/', 'API\PricingController::index');
    $routes->post('/', 'API\PricingController::create');
    $routes->put('(:num)', 'API\PricingController::update/$1');
    $routes->get('active', 'API\PricingController::getActive');
    $routes->post('calculate', 'API\PricingController::calculate');
});
```

### 2. Filtro de Autenticação

**Arquivo: `backend/app/Filters/AuthFilter.php`**

```php
<?php

namespace App\Filters;

use CodeIgniter\Filters\FilterInterface;
use CodeIgniter\HTTP\RequestInterface;
use CodeIgniter\HTTP\ResponseInterface;
use App\Libraries\JWTHandler;

class AuthFilter implements FilterInterface
{
    public function before(RequestInterface $request, $arguments = null)
    {
        $authHeader = $request->getHeaderLine('Authorization');
        
        if (!$authHeader || !preg_match('/Bearer\s+(.*)$/i', $authHeader, $matches)) {
            return service('response')
                ->setStatusCode(401)
                ->setJSON([
                    'success' => false,
                    'message' => 'Token não fornecido'
                ]);
        }
        
        $token = $matches[1];
        $jwt = new JWTHandler();
        
        try {
            $decoded = $jwt->decode($token);
            
            // Adicionar dados do usuário à request
            $request->user = [
                'id' => $decoded->user_id,
                'company_id' => $decoded->company_id,
                'role' => $decoded->role
            ];
            
        } catch (\Exception $e) {
            return service('response')
                ->setStatusCode(401)
                ->setJSON([
                    'success' => false,
                    'message' => 'Token inválido ou expirado'
                ]);
        }
    }

    public function after(RequestInterface $request, ResponseInterface $response, $arguments = null)
    {
        // Não precisa fazer nada aqui
    }
}
```

### 3. JWT Handler

**Arquivo: `backend/app/Libraries/JWTHandler.php`**

```php
<?php

namespace App\Libraries;

use Firebase\JWT\JWT;
use Firebase\JWT\Key;

class JWTHandler
{
    private $key;
    
    public function __construct()
    {
        $this->key = getenv('JWT_SECRET') ?: 'sua-chave-secreta-muito-segura-aqui';
    }
    
    public function encode($data)
    {
        $payload = [
            'iss' => base_url(),
            'iat' => time(),
            'exp' => time() + (60 * 60 * 24), // 24 horas
            'user_id' => $data['id'],
            'company_id' => $data['company_id'],
            'role' => $data['role']
        ];
        
        return JWT::encode($payload, $this->key, 'HS256');
    }
    
    public function decode($token)
    {
        return JWT::decode($token, new Key($this->key, 'HS256'));
    }
}
```

### 4. Controller de Autenticação

**Arquivo: `backend/app/Controllers/API/AuthController.php`**

```php
<?php

namespace App\Controllers\API;

use CodeIgniter\RESTful\ResourceController;
use App\Models\UserModel;
use App\Models\CompanyModel;
use App\Libraries\JWTHandler;

class AuthController extends ResourceController
{
    protected $format = 'json';

    public function login()
    {
        $rules = [
            'email' => 'required|valid_email',
            'password' => 'required|min_length[6]'
        ];
        
        if (!$this->validate($rules)) {
            return $this->fail($this->validator->getErrors(), 400);
        }
        
        $email = $this->request->getVar('email');
        $password = $this->request->getVar('password');
        
        $userModel = new UserModel();
        $user = $userModel->where('email', $email)->first();
        
        if (!$user) {
            return $this->fail('Credenciais inválidas', 401);
        }
        
        if (!password_verify($password, $user['password'])) {
            return $this->fail('Credenciais inválidas', 401);
        }
        
        if ($user['status'] != 'active') {
            return $this->fail('Usuário inativo', 403);
        }
        
        // Atualizar último login
        $userModel->update($user['id'], ['last_login' => date('Y-m-d H:i:s')]);
        
        // Gerar token
        $jwt = new JWTHandler();
        $token = $jwt->encode($user);
        
        // Buscar empresa
        $companyModel = new CompanyModel();
        $company = $companyModel->find($user['company_id']);
        
        unset($user['password']);
        
        return $this->respond([
            'success' => true,
            'message' => 'Login realizado com sucesso',
            'data' => [
                'user' => $user,
                'company' => $company,
                'token' => $token
            ]
        ]);
    }

    public function register()
    {
        $rules = [
            'company_name' => 'required|min_length[3]',
            'cnpj' => 'required|exact_length[14]|is_unique[companies.cnpj]',
            'company_email' => 'required|valid_email',
            'name' => 'required|min_length[3]',
            'email' => 'required|valid_email|is_unique[users.email]',
            'password' => 'required|min_length[6]'
        ];
        
        if (!$this->validate($rules)) {
            return $this->fail($this->validator->getErrors(), 400);
        }
        
        $db = \Config\Database::connect();
        $db->transStart();
        
        try {
            // Criar empresa
            $companyModel = new CompanyModel();
            $companyData = [
                'name' => $this->request->getVar('company_name'),
                'cnpj' => $this->request->getVar('cnpj'),
                'email' => $this->request->getVar('company_email'),
                'phone' => $this->request->getVar('company_phone'),
                'type' => $this->request->getVar('company_type') ?? 'both',
                'api_key' => bin2hex(random_bytes(32)),
                'status' => 'active'
            ];
            
            $companyModel->insert($companyData);
            $companyId = $companyModel->getInsertID();
            
            // Criar usuário admin
            $userModel = new UserModel();
            $userData = [
                'company_id' => $companyId,
                'name' => $this->request->getVar('name'),
                'email' => $this->request->getVar('email'),
                'password' => password_hash($this->request->getVar('password'), PASSWORD_DEFAULT),
                'role' => 'admin',
                'status' => 'active'
            ];
            
            $userModel->insert($userData);
            
            $db->transComplete();
            
            if ($db->transStatus() === false) {
                return $this->fail('Erro ao criar conta', 500);
            }
            
            return $this->respondCreated([
                'success' => true,
                'message' => 'Conta criada com sucesso'
            ]);
            
        } catch (\Exception $e) {
            $db->transRollback();
            return $this->fail('Erro ao criar conta: ' . $e->getMessage(), 500);
        }
    }

    public function me()
    {
        $user = $this->request->user;
        
        $userModel = new UserModel();
        $companyModel = new CompanyModel();
        
        $userData = $userModel->find($user['id']);
        $companyData = $companyModel->find($user['company_id']);
        
        unset($userData['password']);
        
        return $this->respond([
            'success' => true,
            'data' => [
                'user' => $userData,
                'company' => $companyData
            ]
        ]);
    }

    public function logout()
    {
        return $this->respond([
            'success' => true,
            'message' => 'Logout realizado com sucesso'
        ]);
    }
}
```

### 5. Controller de Tickets

**Arquivo: `backend/app/Controllers/API/TicketController.php`**

```php
<?php

namespace App\Controllers\API;

use CodeIgniter\RESTful\ResourceController;
use App\Models\TicketModel;
use App\Models\TransactionModel;
use App\Libraries\PricingCalculator;

class TicketController extends ResourceController
{
    protected $format = 'json';

    public function index()
    {
        $user = $this->request->user;
        $ticketModel = new TicketModel();
        
        $perPage = $this->request->getVar('per_page') ?? 20;
        $status = $this->request->getVar('status');
        
        $builder = $ticketModel->select('tickets.*, 
                                        oc.name as origin_company_name,
                                        dc.name as destination_company_name,
                                        u.name as created_by_name')
                              ->join('companies oc', 'oc.id = tickets.origin_company_id')
                              ->join('companies dc', 'dc.id = tickets.destination_company_id')
                              ->join('users u', 'u.id = tickets.origin_user_id')
                              ->where('(tickets.origin_company_id = ' . $user['company_id'] . 
                                     ' OR tickets.destination_company_id = ' . $user['company_id'] . ')');
        
        if ($status) {
            $builder->where('tickets.status', $status);
        }
        
        $tickets = $builder->orderBy('tickets.created_at', 'DESC')
                          ->paginate($perPage);
        
        return $this->respond([
            'success' => true,
            'data' => $tickets,
            'pager' => $ticketModel->pager->getDetails()
        ]);
    }

    public function show($id)
    {
        $user = $this->request->user;
        $ticketModel = new TicketModel();
        
        $ticket = $ticketModel->select('tickets.*, 
                                       oc.name as origin_company_name,
                                       dc.name as destination_company_name,
                                       u.name as created_by_name')
                             ->join('companies oc', 'oc.id = tickets.origin_company_id')
                             ->join('companies dc', 'dc.id = tickets.destination_company_id')
                             ->join('users u', 'u.id = tickets.origin_user_id')
                             ->where('tickets.id', $id)
                             ->first();
        
        if (!$ticket) {
            return $this->failNotFound('Bilhete não encontrado');
        }
        
        // Verificar permissão
        if ($ticket['origin_company_id'] != $user['company_id'] && 
            $ticket['destination_company_id'] != $user['company_id']) {
            return $this->failUnauthorized('Sem permissão para visualizar este bilhete');
        }
        
        // Buscar validações
        $validationModel = model('ValidationModel');
        $validations = $validationModel->where('ticket_id', $id)->findAll();
        $ticket['validations'] = $validations;
        
        return $this->respond([
            'success' => true,
            'data' => $ticket
        ]);
    }

    public function create()
    {
        $user = $this->request->user;
        $data = $this->request->getJSON(true);
        
        // Validação
        $rules = [
            'destination_company_id' => 'required|integer',
            'box_count' => 'required|integer|greater_than[0]',
            'total_weight' => 'required|decimal|greater_than[0]',
            'total_volume' => 'required|decimal|greater_than[0]',
        ];
        
        if (!$this->validate($rules)) {
            return $this->fail($this->validator->getErrors(), 400);
        }
        
        // Gerar código único
        $ticketCode = $this->generateUniqueCode();
        
        // Calcular valores
        $calculator = new PricingCalculator();
        $pricing = $calculator->calculate(
            $data['total_weight'],
            $data['total_volume'],
            $data['box_count']
        );
        
        // Preparar dados do bilhete
        $ticketData = [
            'ticket_code' => $ticketCode,
            'origin_company_id' => $user['company_id'],
            'origin_user_id' => $user['id'],
            'origin_address' => $data['origin_address'] ?? null,
            'destination_company_id' => $data['destination_company_id'],
            'destination_address' => $data['destination_address'] ?? null,
            'destination_contact' => $data['destination_contact'] ?? null,
            'box_count' => $data['box_count'],
            'total_weight' => $data['total_weight'],
            'total_volume' => $data['total_volume'],
            'dimensions' => json_encode($data['dimensions'] ?? []),
            'cargo_description' => $data['cargo_description'] ?? null,
            'special_instructions' => $data['special_instructions'] ?? null,
            'transport_value' => $pricing['transport_value'],
            'saas_fee' => $pricing['saas_fee'],
            'total_value' => $pricing['total_value'],
            'status' => 'pending',
            'estimated_delivery' => $data['estimated_delivery'] ?? null,
            'metadata' => json_encode($data['metadata'] ?? [])
        ];
        
        $ticketModel = new TicketModel();
        
        $db = \Config\Database::connect();
        $db->transStart();
        
        try {
            if (!$ticketModel->insert($ticketData)) {
                throw new \Exception('Erro ao inserir bilhete');
            }
            
            $ticketId = $ticketModel->getInsertID();
            
            // Gerar QR Code
            $qrCode = $this->generateQRCode($ticketCode);
            $ticketModel->update($ticketId, ['qr_code' => $qrCode]);
            
            // Criar transação
            $this->createTransaction($ticketId, $pricing, $user['company_id']);
            
            $db->transComplete();
            
            if ($db->transStatus() === false) {
                throw new \Exception('Erro na transação');
            }
            
            $ticket = $ticketModel->find($ticketId);
            
            return $this->respondCreated([
                'success' => true,
                'message' => 'Bilhete criado com sucesso',
                'data' => $ticket
            ]);
            
        } catch (\Exception $e) {
            $db->transRollback();
            return $this->fail('Erro ao criar bilhete: ' . $e->getMessage(), 500);
        }
    }

    public function getByCode($code)
    {
        $ticketModel = new TicketModel();
        
        $ticket = $ticketModel->select('tickets.*, 
                                       oc.name as origin_company_name,
                                       dc.name as destination_company_name')
                             ->join('companies oc', 'oc.id = tickets.origin_company_id')
                             ->join('companies dc', 'dc.id = tickets.destination_company_id')
                             ->where('tickets.ticket_code', $code)
                             ->first();
        
        if (!$ticket) {
            return $this->failNotFound('Bilhete não encontrado');
        }
        
        return $this->respond([
            'success' => true,
            'data' => $ticket
        ]);
    }

    public function cancel($id)
    {
        $user = $this->request->user;
        $data = $this->request->getJSON(true);
        
        $ticketModel = new TicketModel();
        $ticket = $ticketModel->find($id);
        
        if (!$ticket) {
            return $this->failNotFound('Bilhete não encontrado');
        }
        
        if ($ticket['origin_company_id'] != $user['company_id']) {
            return $this->failUnauthorized('Apenas a origem pode cancelar o bilhete');
        }
        
        if ($ticket['status'] == 'validated') {
            return $this->fail('Bilhete já validado não pode ser cancelado', 400);
        }
        
        $updateData = [
            'status' => 'cancelled',
            'cancelled_at' => date('Y-m-d H:i:s'),
            'cancellation_reason' => $data['reason'] ?? 'Não informado'
        ];
        
        if ($ticketModel->update($id, $updateData)) {
            // Cancelar transação
            $transactionModel = new TransactionModel();
            $transactionModel->where('ticket_id', $id)->set(['status' => 'cancelled'])->update();
            
            return $this->respond([
                'success' => true,
                'message' => 'Bilhete cancelado com sucesso'
            ]);
        }
        
        return $this->fail('Erro ao cancelar bilhete', 500);
    }

    public function stats()
    {
        $user = $this->request->user;
        $ticketModel = new TicketModel();
        
        $stats = [
            'total' => $ticketModel->where('origin_company_id', $user['company_id'])
                                  ->orWhere('destination_company_id', $user['company_id'])
                                  ->countAllResults(),
            'pending' => $ticketModel->where('status', 'pending')
                                    ->where('(origin_company_id = ' . $user['company_id'] . 
                                           ' OR destination_company_id = ' . $user['company_id'] . ')')
                                    ->countAllResults(),
            'in_transit' => $ticketModel->where('status', 'in_transit')
                                       ->where('(origin_company_id = ' . $user['company_id'] . 
                                              ' OR destination_company_id = ' . $user['company_id'] . ')')
                                       ->countAllResults(),
            'validated' => $ticketModel->where('status', 'validated')
                                      ->where('(origin_company_id = ' . $user['company_id'] . 
                                             ' OR destination_company_id = ' . $user['company_id'] . ')')
                                      ->countAllResults(),
        ];
        
        return $this->respond([
            'success' => true,
            'data' => $stats
        ]);
    }

    private function generateUniqueCode()
    {
        do {
            $code = 'TKT-' . strtoupper(uniqid()) . '-' . strtoupper(bin2hex(random_bytes(3)));
            $ticketModel = new TicketModel();
            $exists = $ticketModel->where('ticket_code', $code)->first();
        } while ($exists);
        
        return $code;
    }

    private function generateQRCode($code)
    {
        // Implementar com biblioteca chillerlan/php-qrcode
        // Por enquanto, retorna base64 simulado
        return 'data:image/png;base64,' . base64_encode("QR-CODE-{$code}");
    }

    private function createTransaction($ticketId, $pricing, $companyId)
    {
        $transactionModel = new TransactionModel();
        
        $transactionModel->insert([
            'ticket_id' => $ticketId,
            'transport_amount' => $pricing['transport_value'],
            'saas_amount' => $pricing['saas_fee'],
            'total_amount' => $pricing['total_value'],
            'payer_company_id' => $companyId,
            'status' => 'pending',
            'due_date' => date('Y-m-d', strtotime('+30 days'))
        ]);
    }
}
```

### 6. Controller de Validação

**Arquivo: `backend/app/Controllers/API/ValidationController.php`**

```php
<?php

namespace App\Controllers\API;

use CodeIgniter\RESTful\ResourceController;
use App\Models\TicketModel;
use App\Models\ValidationModel;
use App\Models\TransactionModel;

class ValidationController extends ResourceController
{
    protected $format = 'json';

    public function validate()
    {
        $user = $this->request->user;
        $data = $this->request->getJSON(true);
        
        // Validação dos dados
        $rules = [
            'ticket_code' => 'required|string',
            'validated_weight' => 'required|decimal',
            'validated_volume' => 'required|decimal',
            'validated_box_count' => 'required|integer'
        ];
        
        if (!$this->validate($rules)) {
            return $this->fail($this->validator->getErrors(), 400);
        }
        
        $ticketModel = new TicketModel();
        $ticket = $ticketModel->where('ticket_code', $data['ticket_code'])->first();
        
        if (!$ticket) {
            return $this->failNotFound('Bilhete não encontrado');
        }
        
        // Verificar se o destino corresponde
        if ($ticket['destination_company_id'] != $user['company_id']) {
            return $this->failUnauthorized('Este bilhete não pertence à sua empresa');
        }
        
        if ($ticket['status'] == 'validated') {
            return $this->fail('Bilhete já foi validado', 400);
        }
        
        if ($ticket['status'] == 'cancelled') {
            return $this->fail('Bilhete está cancelado', 400);
        }
        
        // Calcular diferenças
        $weightDiff = floatval($data['validated_weight']) - floatval($ticket['total_weight']);
        $volumeDiff = floatval($data['validated_volume']) - floatval($ticket['total_volume']);
        $boxDiff = intval($data['validated_box_count']) - intval($ticket['box_count']);
        
        // Determinar conformidade
        $conformity = 'total';
        if (abs($weightDiff) > 1 || abs($volumeDiff) > 0.01 || $boxDiff != 0) {
            $conformity = 'partial';
        }
        if (abs($weightDiff) > 5 || abs($volumeDiff) > 0.1 || abs($boxDiff) > 1) {
            $conformity = 'non_conforming';
        }
        
        $db = \Config\Database::connect();
        $db->transStart();
        
        try {
            // Criar validação
            $validationModel = new ValidationModel();
            $validationData = [
                'ticket_id' => $ticket['id'],
                'validator_user_id' => $user['id'],
                'validated_weight' => $data['validated_weight'],
                'validated_volume' => $data['validated_volume'],
                'validated_box_count' => $data['validated_box_count'],
                'weight_difference' => $weightDiff,
                'volume_difference' => $volumeDiff,
                'box_count_difference' => $boxDiff,
                'notes' => $data['notes'] ?? null,
                'latitude' => $data['latitude'] ?? null,
                'longitude' => $data['longitude'] ?? null,
                'photos' => json_encode($data['photos'] ?? []),
                'signature' => $data['signature'] ?? null,
                'damage_report' => $data['damage_report'] ?? null,
                'conformity' => $conformity
            ];
            
            if (!$validationModel->insert($validationData)) {
                throw new \Exception('Erro ao criar validação');
            }
            
            // Atualizar ticket
            $ticketModel->update($ticket['id'], [
                'status' => 'validated',
                'validated_at' => date('Y-m-d H:i:s')
            ]);
            
            // Atualizar transação para pago
            $transactionModel = new TransactionModel();
            $transactionModel->where('ticket_id', $ticket['id'])
                           ->set([
                               'status' => 'paid',
                               'paid_at' => date('Y-m-d H:i:s'),
                               'payment_method' => 'validation'
                           ])
                           ->update();
            
            $db->transComplete();
            
            if ($db->transStatus() === false) {
                throw new \Exception('Erro na transação');
            }
            
            return $this->respondCreated([
                'success' => true,
                'message' => 'Bilhete validado com sucesso',
                'data' => [
                    'validation' => $validationData,
                    'conformity' => $conformity,
                    'differences' => [
                        'weight' => round($weightDiff, 3),
                        'volume' => round($volumeDiff, 4),
                        'boxes' => $boxDiff
                    ]
                ]
            ]);
            
        } catch (\Exception $e) {
            $db->transRollback();
            return $this->fail('Erro ao validar bilhete: ' . $e->getMessage(), 500);
        }
    }

    public function getByTicket($ticketId)
    {
        $validationModel = new ValidationModel();
        
        $validations = $validationModel->select('validations.*, u.name as validator_name')
                                      ->join('users u', 'u.id = validations.validator_user_id')
                                      ->where('validations.ticket_id', $ticketId)
                                      ->findAll();
        
        return $this->respond([
            'success' => true,
            'data' => $validations
        ]);
    }

    public function history()
    {
        $user = $this->request->user;
        $validationModel = new ValidationModel();
        
        $validations = $validationModel->select('validations.*, 
                                                t.ticket_code,
                                                t.origin_company_id,
                                                t.destination_company_id,
                                                u.name as validator_name')
                                      ->join('tickets t', 't.id = validations.ticket_id')
                                      ->join('users u', 'u.id = validations.validator_user_id')
                                      ->where('validations.validator_user_id', $user['id'])
                                      ->orderBy('validations.validated_at', 'DESC')
                                      ->paginate(20);
        
        return $this->respond([
            'success' => true,
            'data' => $validations,
            'pager' => $validationModel->pager->getDetails()
        ]);
    }
}
```

### 7. Calculadora de Preços

**Arquivo: `backend/app/Libraries/PricingCalculator.php`**

```php
<?php

namespace App\Libraries;

use App\Models\PricingModel;

class PricingCalculator
{
    public function calculate($weight, $volume, $boxCount, $options = [])
    {
        $pricingModel = new PricingModel();
        
        // Buscar regra ativa
        $rule = $pricingModel->where('is_active', true)
                            ->where('is_default', true)
                            ->first();
        
        if (!$rule) {
            // Buscar qualquer regra ativa
            $rule = $pricingModel->where('is_active', true)
                                ->orderBy('id', 'DESC')
                                ->first();
        }
        
        if (!$rule) {
            throw new \Exception('Nenhuma regra de precificação ativa encontrada');
        }
        
        // Cálculo base
        $transportValue = floatval($rule['base_price']);
        $transportValue += (floatval($weight) * floatval($rule['price_per_kg']));
        $transportValue += (floatval($volume) * floatval($rule['price_per_m3']));
        $transportValue += (intval($boxCount) * floatval($rule['price_per_box']));
        
        // Aplicar descontos por faixa
        $discount = $this->calculateTierDiscount($transportValue, $weight, $volume, $rule);
        $transportValue -= $discount;
        
        // Taxas adicionais
        $additionalFees = 0;
        if (isset($options['fragile']) && $options['fragile']) {
            $additionalFees += floatval($rule['fragile_cargo_fee']);
        }
        if (isset($options['express']) && $options['express']) {
            $additionalFees += floatval($rule['express_delivery_fee']);
        }
        
        $transportValue += $additionalFees;
        
        // Calcular taxa SaaS
        $saasFee = ($transportValue * floatval($rule['saas_percentage']) / 100);
        $saasFee += floatval($rule['saas_fixed_fee']);
        
        // Total
        $totalValue = $transportValue + $saasFee;
        
        return [
            'transport_value' => round($transportValue, 2),
            'saas_fee' => round($saasFee, 2),
            'total_value' => round($totalValue, 2),
            'discount_applied' => round($discount, 2),
            'additional_fees' => round($additionalFees, 2),
            'rule_applied' => $rule['name'],
            'breakdown' => [
                'base' => floatval($rule['base_price']),
                'weight' => round(floatval($weight) * floatval($rule['price_per_kg']), 2),
                'volume' => round(floatval($volume) * floatval($rule['price_per_m3']), 2),
                'boxes' => round(intval($boxCount) * floatval($rule['price_per_box']), 2)
            ]
        ];
    }
    
    private function calculateTierDiscount($baseValue, $weight, $volume, $rule)
    {
        $totalDiscount = 0;
        
        // Descontos por peso
        if (!empty($rule['weight_tiers'])) {
            $weightTiers = json_decode($rule['weight_tiers'], true);
            if ($weightTiers) {
                foreach ($weightTiers as $tier) {
                    if ($weight >= $tier['min'] && $weight <= $tier['max']) {
                        $totalDiscount += ($baseValue * floatval($tier['discount_percentage']) / 100);
                        break;
                    }
                }
            }
        }
        
        // Descontos por volume
        if (!empty($rule['volume_tiers'])) {
            $volumeTiers = json_decode($rule['volume_tiers'], true);
            if ($volumeTiers) {
                foreach ($volumeTiers as $tier) {
                    if ($volume >= $tier['min'] && $volume <= $tier['max']) {
                        $totalDiscount += ($baseValue * floatval($tier['discount_percentage']) / 100);
                        break;
                    }
                }
            }
        }
        
        return $totalDiscount;
    }
}
```

### 8. Models

**Arquivo: `backend/app/Models/TicketModel.php`**

```php
<?php

namespace App\Models;

use CodeIgniter\Model;

class TicketModel extends Model
{
    protected $table = 'tickets';
    protected $primaryKey = 'id';
    protected $useAutoIncrement = true;
    protected $returnType = 'array';
    protected $useSoftDeletes = false;
    protected $protectFields = true;
    
    protected $allowedFields = [
        'ticket_code', 'qr_code', 'origin_company_id', 'origin_user_id', 
        'origin_address', 'destination_company_id', 'destination_address', 
        'destination_contact', 'box_count', 'total_weight', 'total_volume', 
        'dimensions', 'cargo_description', 'special_instructions', 'status', 
        'transport_value', 'saas_fee', 'total_value', 'issued_at', 
        'estimated_delivery', 'validated_at', 'cancelled_at', 
        'cancellation_reason', 'metadata'
    ];

    protected $useTimestamps = true;
    protected $createdField = 'created_at';
    protected $updatedField = 'updated_at';

    protected $validationRules = [
        'origin_company_id' => 'required|integer',
        'destination_company_id' => 'required|integer',
        'box_count' => 'required|integer|greater_than[0]',
        'total_weight' => 'required|decimal',
        'total_volume' => 'required|decimal'
    ];
}
```

**Arquivo: `backend/app/Models/ValidationModel.php`**

```php
<?php

namespace App\Models;

use CodeIgniter\Model;

class ValidationModel extends Model
{
    protected $table = 'validations';
    protected $primaryKey = 'id';
    protected $returnType = 'array';
    protected $useSoftDeletes = false;
    
    protected $allowedFields = [
        'ticket_id', 'validator_user_id', 'validated_weight', 
        'validated_volume', 'validated_box_count', 'weight_difference', 
        'volume_difference', 'box_count_difference', 'notes', 'photos', 
        'signature', 'latitude', 'longitude', 'damage_report', 
        'conformity', 'validated_at'
    ];

    protected $useTimestamps = false;
    
    protected $validationRules = [
        'ticket_id' => 'required|integer',
        'validator_user_id' => 'required|integer',
        'validated_weight' => 'required|decimal',
        'validated_volume' => 'required|decimal'
    ];
}
```

**Demais Models seguem padrão similar (UserModel, CompanyModel, TransactionModel, PricingModel)**

---

## ⚛️ Frontend - React

### 1. Configuração da API

**Arquivo: `frontend/src/services/api.js`**

```javascript
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:8080/api';

const api = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Interceptor para adicionar token
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Interceptor para tratar erros
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      localStorage.removeItem('user');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

// ============================================
// AUTH API
// ============================================
export const authAPI = {
  login: (credentials) => api.post('/auth/login', credentials),
  register: (data) => api.post('/auth/register', data),
  me: () => api.get('/auth/me'),
  logout: () => api.post('/auth/logout'),
};

// ============================================
// TICKETS API
// ============================================
export const ticketAPI = {
  getAll: (params) => api.get('/tickets', { params }),
  getById: (id) => api.get(`/tickets/${id}`),
  getByCode: (code) => api.get(`/tickets/code/${code}`),
  create: (data) => api.post('/tickets', data),
  update: (id, data) => api.put(`/tickets/${id}`, data),
  cancel: (id, reason) => api.delete(`/tickets/${id}`, { data: { reason } }),
  getStats: () => api.get('/tickets/stats/overview'),
};

// ============================================
// VALIDATIONS API
// ============================================
export const validationAPI = {
  validate: (data) => api.post('/validations', data),
  getByTicket: (ticketId) => api.get(`/validations/ticket/${ticketId}`),
  getHistory: (params) => api.get('/validations/history', { params }),
};

// ============================================
// BILLING API
// ============================================
export const billingAPI = {
  getDashboard: () => api.get('/billing/dashboard'),
  getTransactions: (params) => api.get('/billing/transactions', { params }),
  getReport: (params) => api.get('/billing/report', { params }),
  markAsPaid: (id) => api.post(`/billing/pay/${id}`),
};

// ============================================
// COMPANIES API
// ============================================
export const companyAPI = {
  getAll: (params) => api.get('/companies', { params }),
  getById: (id) => api.get(`/companies/${id}`),
  search: (query) => api.get('/companies/search', { params: { q: query } }),
};

export default api;
```

### 2. Componente de Criação de Bilhete

**Arquivo: `frontend/src/components/Tickets/CreateTicket.jsx`**

```javascript
import React, { useState, useEffect } from 'react';
import { ticketAPI, companyAPI } from '../../services/api';
import { useNavigate } from 'react-router-dom';

const CreateTicket = () => {
  const navigate = useNavigate();
  
  const [companies, setCompanies] = useState([]);
  const [loading, setLoading] = useState(false);
  
  const [formData, setFormData] = useState({
    destination_company_id: '',
    destination_address: '',
    destination_contact: '',
    cargo_description: '',
    special_instructions: '',
    estimated_delivery: '',
  });

  const [boxes, setBoxes] = useState([{
    width: '',
    height: '',
    depth: '',
    weight: '',
    description: ''
  }]);

  const [totals, setTotals] = useState({
    box_count: 0,
    total_weight: 0,
    total_volume: 0
  });

  useEffect(() => {
    loadCompanies();
  }, []);

  useEffect(() => {
    calculateTotals();
  }, [boxes]);

  const loadCompanies = async () => {
    try {
      const response = await companyAPI.getAll();
      if (response.data.success) {
        setCompanies(response.data.data);
      }
    } catch (error) {
      console.error('Erro ao carregar empresas:', error);
    }
  };

  const handleBoxChange = (index, field, value) => {
    const newBoxes = [...boxes];
    newBoxes[index][field] = value;
    setBoxes(newBoxes);
  };

  const addBox = () => {
    setBoxes([...boxes, { 
      width: '', 
      height: '', 
      depth: '', 
      weight: '',
      description: ''
    }]);
  };

  const removeBox = (index) => {
    if (boxes.length > 1) {
      const newBoxes = boxes.filter((_, i) => i !== index);
      setBoxes(newBoxes);
    }
  };

  const calculateTotals = () => {
    let totalWeight = 0;
    let totalVolume = 0;

    boxes.forEach(box => {
      if (box.weight) totalWeight += parseFloat(box.weight);
      
      if (box.width && box.height && box.depth) {
        // Converter cm³ para m³
        const volume = (parseFloat(box.width) * parseFloat(box.height) * parseFloat(box.depth)) / 1000000;
        totalVolume += volume;
      }
    });

    setTotals({
      box_count: boxes.length,
      total_weight: totalWeight.toFixed(3),
      total_volume: totalVolume.toFixed(4)
    });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    if (!totals.total_weight || !totals.total_volume) {
      alert('Por favor, preencha todos os dados das caixas');
      return;
    }

    setLoading(true);

    try {
      const ticketData = {
        ...formData,
        box_count: totals.box_count,
        total_weight: parseFloat(totals.total_weight),
        total_volume: parseFloat(totals.total_volume),
        dimensions: boxes
      };

      const response = await ticketAPI.create(ticketData);
      
      if (response.data.success) {
        alert('Bilhete criado com sucesso!');
        navigate('/tickets');
      }
    } catch (error) {
      const message = error.response?.data?.message || error.message;
      alert('Erro ao criar bilhete: ' + message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="container mx-auto px-4 py-6">
      <div className="max-w-5xl mx-auto">
        <div className="bg-white rounded-lg shadow-lg p-6">
          <h2 className="text-3xl font-bold mb-6 text-gray-800">
            📦 Emitir Novo Bilhete
          </h2>
          
          <form onSubmit={handleSubmit} className="space-y-8">
            {/* SEÇÃO: Destino */}
            <div className="border-b pb-6">
              <h3 className="text-xl font-semibold mb-4 text-gray-700 flex items-center">
                <span className="mr-2">🎯</span> Informações de Destino
              </h3>
              
              <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                <div className="md:col-span-2">
                  <label className="block text-sm font-medium mb-2 text-gray-700">
                    Empresa Destino *
                  </label>
                  <select
                    className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
                    value={formData.destination_company_id}
                    onChange={(e) => setFormData({...formData, destination_company_id: e.target.value})}
                    required
                  >
                    <option value="">Selecione uma empresa...</option>
                    {companies.map(company => (
                      <option key={company.id} value={company.id}>
                        {company.name} - {company.cnpj}
                      </option>
                    ))}
                  </select>
                </div>

                <div className="md:col-span-2">
                  <label className="block text-sm font-medium mb-2 text-gray-700">
                    Endereço de Entrega
                  </label>
                  <input
                    type="text"
                    className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                    value={formData.destination_address}
                    onChange={(e) => setFormData({...formData, destination_address: e.target.value})}
                    placeholder="Rua, número, bairro, cidade..."
                  />
                </div>

                <div>
                  <label className="block text-sm font-medium mb-2 text-gray-700">
                    Contato no Destino
                  </label>
                  <input
                    type="text"
                    className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                    value={formData.destination_contact}
                    onChange={(e) => setFormData({...formData, destination_contact: e.target.value})}
                    placeholder="Nome do responsável"
                  />
                </div>

                <div>
                  <label className="block text-sm font-medium mb-2 text-gray-700">
                    Previsão de Entrega
                  </label>
                  <input
                    type="datetime-local"
                    className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                    value={formData.estimated_delivery}
                    onChange={(e) => setFormData({...formData, estimated_delivery: e.target.value})}
                  />
                </div>
              </div>
            </div>

            {/* SEÇÃO: Caixas */}
            <div className="border-b pb-6">
              <div className="flex justify-between items-center mb-4">
                <h3 className="text-xl font-semibold text-gray-700 flex items-center">
                  <span className="mr-2">📦</span> Caixas ({boxes.length})
                </h3>
                <button
                  type="button"
                  onClick={addBox}
                  className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition flex items-center"
                >
                  <span className="mr-2">+</span> Adicionar Caixa
                </button>
              </div>

              <div className="space-y-4">
                {boxes.map((box, index) => (
                  <div key={index} className="bg-gray-50 p-4 rounded-lg border border-gray-200">
                    <div className="flex justify-between items-center mb-3">
                      <span className="font-semibold text-gray-700">Caixa #{index + 1}</span>
                      {boxes.length > 1 && (
                        <button
                          type="button"
                          onClick={() => removeBox(index)}
                          className="text-red-600 hover:text-red-800 font-medium"
                        >
                          ✕ Remover
                        </button>
                      )}
                    </div>

                    <div className="grid grid-cols-2 md:grid-cols-5 gap-3">
                      <div>
                        <label className="block text-xs font-medium mb-1 text-gray-600">
                          Largura (cm) *
                        </label>
                        <input
                          type="number"
                          step="0.01"
                          className="w-full p-2 border border-gray-300 rounded text-sm focus:ring-2 focus:ring-blue-500"
                          value={box.width}
                          onChange={(e) => handleBoxChange(index, 'width', e.target.value)}
                          required
                        />
                      </div>
                      <div>
                        <label className="block text-xs font-medium mb-1 text-gray-600">
                          Altura (cm) *
                        </label>
                        <input
                          type="number"
                          step="0.01"
                          className="w-full p-2 border border-gray-300 rounded text-sm focus:ring-2 focus:ring-blue-500"
                          value={box.height}
                          onChange={(e) => handleBoxChange(index, 'height', e.target.value)}
                          required
                        />
                      </div>
                      <div>
                        <label className="block text-xs font-medium mb-1 text-gray-600">
                          Profundidade (cm) *
                        </label>
                        <input
                          type="number"
                          step="0.01"
                          className="w-full p-2 border border-gray-300 rounded text-sm focus:ring-2 focus:ring-blue-500"
                          value={box.depth}
                          onChange={(e) => handleBoxChange(index, 'depth', e.target.value)}
                          required
                        />
                      </div>
                      <div>
                        <label className="block text-xs font-medium mb-1 text-gray-600">
                          Peso (kg) *
                        </label>
                        <input
                          type="number"
                          step="0.001"
                          className="w-full p-2 border border-gray-300 rounded text-sm focus:ring-2 focus:ring-blue-500"
                          value={box.weight}
                          onChange={(e) => handleBoxChange(index, 'weight', e.target.value)}
                          required
                        />
                      </div>
                      <div className="col-span-2 md:col-span-1">
                        <label className="block text-xs font-medium mb-1 text-gray-600">
                          Descrição
                        </label>
                        <input
                          type="text"
                          className="w-full p-2 border border-gray-300 rounded text-sm focus:ring-2 focus:ring-blue-500"
                          value={box.description}
                          onChange={(e) => handleBoxChange(index, 'description', e.target.value)}
                          placeholder="Ex: Eletrônicos"
                        />
                      </div>
                    </div>
                  </div>
                ))}
              </div>
            </div>

            {/* SEÇÃO: Descrição da Carga */}
            <div className="border-b pb-6">
              <h3 className="text-xl font-semibold mb-4 text-gray-700">
                📝 Descrição da Carga
              </h3>
              
              <div className="space-y-4">
                <div>
                  <label className="block text-sm font-medium mb-2 text-gray-700">
                    Descrição Geral
                  </label>
                  <textarea
                    className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                    rows="3"
                    value={formData.cargo_description}
                    onChange={(e) => setFormData({...formData, cargo_description: e.target.value})}
                    placeholder="Descreva o conteúdo da carga..."
                  />
                </div>

                <div>
                  <label className="block text-sm font-medium mb-2 text-gray-700">
                    Instruções Especiais
                  </label>
                  <textarea
                    className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                    rows="2"
                    value={formData.special_instructions}
                    onChange={(e) => setFormData({...formData, special_instructions: e.target.value})}
                    placeholder="Ex: Carga frágil, manter refrigerado..."
                  />
                </div>
              </div>
            </div>

            {/* SEÇÃO: Resumo */}
            <div className="bg-gradient-to-r from-blue-50 to-indigo-50 p-6 rounded-lg border border-blue-200">
              <h3 className="text-xl font-semibold mb-4 text-gray-800">
                📊 Resumo do Envio
              </h3>
              <div className="grid grid-cols-3 gap-6">
                <div className="text-center">
                  <p className="text-sm text-gray-600 mb-1">Total de Caixas</p>
                  <p className="text-3xl font-bold text-blue-600">{totals.box_count}</p>
                </div>
                <div className="text-center">
                  <p className="text-sm text-gray-600 mb-1">Peso Total</p>
                  <p className="text-3xl font-bold text-green-600">{totals.total_weight} kg</p>
                </div>
                <div className="text-center">
                  <p className="text-sm text-gray-600 mb-1">Volume Total</p>
                  <p className="text-3xl font-bold text-purple-600">{totals.total_volume} m³</p>
                </div>
              </div>
            </div>

            {/* Botões */}
            <div className="flex justify-end space-x-4 pt-4">
              <button
                type="button"
                className="px-6 py-3 border border-gray-300 rounded-lg hover:bg-gray-50 transition font-medium"
                onClick={() => navigate('/tickets')}
              >
                Cancelar
              </button>
              <button
                type="submit"
                disabled={loading}
                className="px-8 py-3 bg-green-600 text-white rounded-lg hover:bg-green-700 disabled:bg-gray-400 transition font-semibold flex items-center"
              >
                {loading ? (
                  <>
                    <svg className="animate-spin -ml-1 mr-3 h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                      <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                      <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                    </svg>
                    Criando...
                  </>
                ) : (
                  <>
                    <span className="mr-2">✓</span>
                    Emitir Bilhete
                  </>
                )}
              </button>
            </div>
          </form>
        </div>
      </div>
    </div>
  );
};

export default CreateTicket;
```

### 3. Componente de Validação

**Arquivo: `frontend/src/components/Validation/ValidateTicket.jsx`**

```javascript
import React, { useState } from 'react';
import { validationAPI, ticketAPI } from '../../services/api';
import { Html5QrcodeScanner } from 'html5-qrcode';

const ValidateTicket = () => {
  const [ticketCode, setTicketCode] = useState('');
  const [ticket, setTicket] = useState(null);
  const [showScanner, setShowScanner] = useState(false);
  const [loading, setLoading] = useState(false);
  
  const [validationData, setValidationData] = useState({
    validated_weight: '',
    validated_volume: '',
    validated_box_count: '',
    notes: '',
    damage_report: ''
  });

  const searchTicket = async (code) => {
    if (!code) return;
    
    setLoading(true);
    try {
      const response = await ticketAPI.getByCode(code);
      if (response.data.success) {
        const ticketData = response.data.data;
        setTicket(ticketData);
        setValidationData(prev => ({
          ...prev,
          validated_box_count: ticketData.box_count,
          validated_weight: ticketData.total_weight,
          validated_volume: ticketData.total_volume
        }));
      }
    } catch (error) {
      alert('Bilhete não encontrado: ' + (error.response?.data?.message || error.message));
      setTicket(null);
    } finally {
      setLoading(false);
    }
  };

  const initScanner = () => {
    const scanner = new Html5QrcodeScanner('qr-reader', {
      qrbox: { width: 250, height: 250 },
      fps: 5
    });

    scanner.render((decodedText) => {
      setTicketCode(decodedText);
      searchTicket(decodedText);
      scanner.clear();
      setShowScanner(false);
    });
  };

  const handleValidate = async (e) => {
    e.preventDefault();

    if (!ticket) {
      alert('Nenhum bilhete carregado');
      return;
    }

    setLoading(true);
    try {
      const response = await validationAPI.validate({
        ticket_code: ticketCode,
        ...validationData
      });

      if (response.data.success) {
        const { conformity, differences } = response.data.data;
        
        let message = 'Bilhete validado com sucesso!\n\n';
        
        if (conformity !== 'total') {
          message += '⚠️ DIFERENÇAS DETECTADAS:\n';
          if (Math.abs(differences.weight) > 0.1) {
            message += `Peso: ${differences.weight > 0 ? '+' : ''}${differences.weight.toFixed(3)} kg\n`;
          }
          if (Math.abs(differences.volume) > 0.01) {
            message += `Volume: ${differences.volume > 0 ? '+' : ''}${differences.volume.toFixed(4)} m³\n`;
          }
          if (differences.boxes !== 0) {
            message += `Caixas: ${differences.boxes > 0 ? '+' : ''}${differences.boxes}\n`;
          }
        }
        
        alert(message);
        
        // Resetar
        setTicket(null);
        setTicketCode('');
        setValidationData({
          validated_weight: '',
          validated_volume: '',
          validated_box_count: '',
          notes: '',
          damage_report: ''
        });
      }
    } catch (error) {
      alert('Erro ao validar: ' + (error.response?.data?.message || error.message));
    } finally {
      setLoading(false);
    }
  };

  React.useEffect(() => {
    if (showScanner) {
      initScanner();
    }
  }, [showScanner]);

  return (
    <div className="container mx-auto px-4 py-6">
      <div className="max-w-4xl mx-auto">
        <h2 className="text-3xl font-bold mb-6 text-gray-800">
          ✓ Validar Bilhete
        </h2>

        {/* SCANNER / BUSCA */}
        <div className="bg-white rounded-lg shadow p-6 mb-6">
          <div className="flex flex-col md:flex-row md:items-center space-y-3 md:space-y-0 md:space-x-3 mb-4">
            <input
              type="text"
              className="flex-1 p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
              placeholder="Digite ou escaneie o código do bilhete..."
              value={ticketCode}
              onChange={(e) => setTicketCode(e.target.value)}
              onKeyPress={(e) => e.key === 'Enter' && searchTicket(ticketCode)}
            />
            <button
              onClick={() => searchTicket(ticketCode)}
              disabled={loading}
              className="px-6 py-3 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:bg-gray-400 transition font-medium"
            >
              🔍 Buscar
            </button>
            <button
              onClick={() => setShowScanner(!showScanner)}
              className="px-6 py-3 bg-purple-600 text-white rounded-lg hover:bg-purple-700 transition font-medium"
            >
              📷 Escanear
            </button>
          </div>

          {showScanner && (
            <div className="mt-4 border-2 border-purple-300 rounded-lg p-4 bg-purple-50">
              <div id="qr-reader"></div>
            </div>
          )}
        </div>

        {/* INFORMAÇÕES DO BILHETE */}
        {ticket && (
          <div className="bg-white rounded-lg shadow p-6 mb-6">
            <div className="flex justify-between items-start mb-4">
              <h3 className="text-2xl font-bold text-gray-800">
                Informações do Bilhete
              </h3>
              <span className={`px-4 py-2 rounded-full text-sm font-semibold ${
                ticket.status === 'pending' ? 'bg-yellow-100 text-yellow-800' :
                ticket.status === 'validated' ? 'bg-green-100 text-green-800' :
                ticket.status === 'cancelled' ? 'bg-red-100 text-red-800' :
                'bg-gray-100 text-gray-800'
              }`}>
                {ticket.status === 'pending' ? 'Pendente' :
                 ticket.status === 'validated' ? 'Validado' :
                 ticket.status === 'cancelled' ? 'Cancelado' : ticket.status}
              </span>
            </div>
            
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4 mb-6">
              <div>
                <p className="text-sm text-gray-600">Código</p>
                <p className="font-mono font-semibold text-lg">{ticket.ticket_code}</p>
              </div>
              <div>
                <p className="text-sm text-gray-600">Emitido em</p>
                <p className="font-semibold">{new Date(ticket.issued_at).toLocaleString('pt-BR')}</p>
              </div>
              <div>
                <p className="text-sm text-gray-600">Origem</p>
                <p className="font-semibold">{ticket.origin_company_name}</p>
              </div>
              <div>
                <p className="text-sm text-gray-600">Destino</p>
                <p className="font-semibold">{ticket.destination_company_name}</p>
              </div>
            </div>

            <div className="grid grid-cols-3 gap-4 p-4 bg-blue-50 rounded-lg">
              <div className="text-center">
                <p className="text-sm text-gray-600 mb-1">Caixas Declaradas</p>
                <p className="text-2xl font-bold text-blue-600">{ticket.box_count}</p>
              </div>
              <div className="text-center">
                <p className="text-sm text-gray-600 mb-1">Peso Declarado</p>
                <p className="text-2xl font-bold text-green-600">{ticket.total_weight} kg</p>
              </div>
              <div className="text-center">
                <p className="text-sm text-gray-600 mb-1">Volume Declarado</p>
                <p className="text-2xl font-bold text-purple-600">{ticket.total_volume} m³</p>
              </div>
            </div>

            {ticket.cargo_description && (
              <div className="mt-4 p-3 bg-gray-50 rounded">
                <p className="text-sm text-gray-600 mb-1">Descrição da Carga</p>
                <p className="text-gray-800">{ticket.cargo_description}</p>
              </div>
            )}
          </div>
        )}

        {/* FORMULÁRIO DE VALIDAÇÃO */}
        {ticket && ticket.status === 'pending' && (
          <form onSubmit={handleValidate} className="bg-white rounded-lg shadow p-6">
            <h3 className="text-2xl font-bold mb-6 text-gray-800">
              Conferência da Carga
            </h3>

            <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-6">
              <div>
                <label className="block text-sm font-medium mb-2 text-gray-700">
                  Quantidade de Caixas *
                </label>
                <input
                  type="number"
                  className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                  value={validationData.validated_box_count}
                  onChange={(e) => setValidationData({...validationData, validated_box_count: e.target.value})}
                  required
                />
              </div>
              <div>
                <label className="block text-sm font-medium mb-2 text-gray-700">
                  Peso Real (kg) *
                </label>
                <input
                  type="number"
                  step="0.001"
                  className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                  value={validationData.validated_weight}
                  onChange={(e) => setValidationData({...validationData, validated_weight: e.target.value})}
                  required
                />
              </div>
              <div>
                <label className="block text-sm font-medium mb-2 text-gray-700">
                  Volume Real (m³) *
                </label>
                <input
                  type="number"
                  step="0.0001"
                  className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                  value={validationData.validated_volume}
                  onChange={(e) => setValidationData({...validationData, validated_volume: e.target.value})}
                  required
                />
              </div>
            </div>

            <div className="mb-4">
              <label className="block text-sm font-medium mb-2 text-gray-700">
                Observações
              </label>
              <textarea
                className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                rows="3"
                value={validationData.notes}
                onChange={(e) => setValidationData({...validationData, notes: e.target.value})}
                placeholder="Observações sobre a conferência da carga..."
              />
            </div>

            <div className="mb-6">
              <label className="block text-sm font-medium mb-2 text-gray-700">
                Relato de Avarias (se houver)
              </label>
              <textarea
                className="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
                rows="2"
                value={validationData.damage_report}
                onChange={(e) => setValidationData({...validationData, damage_report: e.target.value})}
                placeholder="Descreva qualquer avaria encontrada..."
              />
            </div>

            <button
              type="submit"
              disabled={loading}
              className="w-full py-4 bg-green-600 text-white rounded-lg hover:bg-green-700 disabled:bg-gray-400 transition font-bold text-lg"
            >
              {loading ? (
                <span className="flex items-center justify-center">
                  <svg className="animate-spin -ml-1 mr-3 h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                    <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                    <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                  </svg>
                  Validando...
                </span>
              ) : (
                '✓ Validar e Finalizar Bilhete'
              )}
            </button>
          </form>
        )}

        {ticket && ticket.status === 'validated' && (
          <div className="bg-green-50 border-2 border-green-500 rounded-lg p-6 text-center">
            <div className="text-6xl mb-4">✓</div>
            <h3 className="text-2xl font-bold text-green-800 mb-2">
              Bilhete Já Validado
            </h3>
            <p className="text-green-700">
              Este bilhete foi validado em {new Date(ticket.validated_at).toLocaleString('pt-BR')}
            </p>
          </div>
        )}
      </div>
    </div>
  );
};

export default ValidateTicket;
```

---

## 🐳 Docker Configuration

**Arquivo: `docker-compose.yml`**

```yaml
version: '3.8'

services:
  # MySQL Database
  mysql:
    image: mysql:8.0
    container_name: logistics_mysql
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: logistics_ticketing
      MYSQL_USER: logistics_user
      MYSQL_PASSWORD: logistics_pass
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - logistics_network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 20s
      retries: 10

  # PHP Backend (CodeIgniter 4)
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: logistics_backend
    ports:
      - "8080:8080"
    volumes:
      - ./backend:/var/www/html
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      DB_HOST: mysql
      DB_DATABASE: logistics_ticketing
      DB_USERNAME: logistics_user
      DB_PASSWORD: logistics_pass
      DB_PORT: 3306
      JWT_SECRET: your-super-secret-jwt-key-change-this-in-production
    networks:
      - logistics_network
    command: php spark serve --host=0.0.0.0 --port=8080

  # React Frontend
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: logistics_frontend
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      REACT_APP_API_URL: http://localhost:8080/api
    networks:
      - logistics_network
    depends_on:
      - backend

  # phpMyAdmin (Opcional - para gerenciar o banco)
  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    container_name: logistics_phpmyadmin
    environment:
      PMA_HOST: mysql
      PMA_PORT: 3306
      PMA_USER: root
      PMA_PASSWORD: root123
    ports:
      - "8081:80"
    depends_on:
      - mysql
    networks:
      - logistics_network

volumes:
  mysql_data:
    driver: local

networks:
  logistics_network:
    driver: bridge
```

**Arquivo: `backend/Dockerfile`**

```dockerfile
FROM php:8.2-cli

# Instalar extensões PHP necessárias
RUN apt-get update && apt-get install -y \
    git \
    unzip \
    libzip-dev \
    && docker-php-ext-install pdo pdo_mysql mysqli zip

# Instalar Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

# Copiar arquivos
COPY . .

# Instalar dependências
RUN composer install --no-dev --optimize-autoloader

# Permissões
RUN chmod -R 755 /var/www/html/writable

EXPOSE 8080

CMD ["php", "spark", "serve", "--host=0.0.0.0", "--port=8080"]
```

**Arquivo: `frontend/Dockerfile`**

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

---

## 🚀 Instruções de Instalação

### 1. Clonar e Preparar o Projeto

```bash
# Criar estrutura de pastas
mkdir logistics-ticketing
cd logistics-ticketing

# Criar subpastas
mkdir -p backend frontend database

# Copiar arquivos SQL
# Colocar o init.sql na pasta database/
```

### 2. Configurar Backend (CodeIgniter 4)

```bash
cd backend

# Instalar CodeIgniter via Composer
composer create-project codeigniter4/appstarter .

# Configurar .env
cp env .env

# Editar .env com as configurações do banco
```

**Editar `backend/.env`:**

```env
CI_ENVIRONMENT = development

database.default.hostname = mysql
database.default.database = logistics_ticketing
database.default.username = logistics_user
database.default.password = logistics_pass
database.default.DBDriver = MySQLi

JWT_SECRET = your-super-secret-jwt-key-change-this-in-production
```

### 3. Configurar Frontend (React)

```bash
cd frontend

# Criar app React
npx create-react-app .

# Instalar dependências
npm install axios react-router-dom html5-qrcode

# Criar .env
```

**Arquivo: `frontend/.env`**

```env
REACT_APP_API_URL=http://localhost:8080/api
```

### 4. Subir os Containers

```bash
# Na raiz do projeto
docker-compose up -d

# Verificar se tudo está rodando
docker-compose ps

# Ver logs
docker-compose logs -f
```

### 5. Acessar a Aplicação

- **Frontend**: http://localhost:3000
- **Backend API**: http://localhost:8080/api
- **phpMyAdmin**: http://localhost:8081 (usuário: root, senha: root123)

---

## 📝 Notas Finais

Este projeto está completo e pronto para desenvolvimento. Todos os códigos estão em um único README para facilitar a cópia e implementação.

**Próximos passos sugeridos:**
1. Implementar autenticação JWT completa
2. Adicionar geração real de QR Code (biblioteca `chillerlan/php-qrcode`)
3. Implementar upload de fotos na validação
4. Criar dashboard com gráficos (Chart.js ou Recharts)
5. Adicionar testes unitários e de integração
6. Configurar CI/CD
7. Implementar notificações por email/SMS
8. Adicionar relatórios em PDF

**Segurança:**
- Trocar `JWT_SECRET` em produção
- Configurar CORS adequadamente
- Implementar rate limiting
- Adicionar validação de CNPJ
- Criptografar dados sensíveis

Qualquer dúvida, é só perguntar! 🚀