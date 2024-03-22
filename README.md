Sistema de Bilhetagem para Logística de Caixas
Vou montar um projeto completo para este sistema SaaS de bilhetagem logística.
📋 Estrutura do Projeto
logistics-ticketing/
├── backend/
│   ├── app/
│   │   ├── Config/
│   │   │   ├── Routes.php
│   │   │   ├── Database.php
│   │   │   └── Filters.php
│   │   ├── Controllers/
│   │   │   ├── API/
│   │   │   │   ├── AuthController.php
│   │   │   │   ├── TicketController.php
│   │   │   │   ├── CompanyController.php
│   │   │   │   ├── ValidationController.php
│   │   │   │   └── BillingController.php
│   │   ├── Models/
│   │   │   ├── CompanyModel.php
│   │   │   ├── TicketModel.php
│   │   │   ├── ValidationModel.php
│   │   │   ├── PricingModel.php
│   │   │   └── TransactionModel.php
│   │   ├── Filters/
│   │   │   └── AuthFilter.php
│   │   └── Libraries/
│   │       ├── JWTHandler.php
│   │       └── PricingCalculator.php
│   └── writable/
├── frontend/
│   ├── public/
│   ├── src/
│   │   ├── components/
│   │   │   ├── Auth/
│   │   │   ├── Tickets/
│   │   │   ├── Validation/
│   │   │   ├── Dashboard/
│   │   │   └── Billing/
│   │   ├── services/
│   │   │   └── api.js
│   │   ├── contexts/
│   │   │   └── AuthContext.js
│   │   ├── utils/
│   │   └── App.js
│   └── package.json
└── docker-compose.yml
🗄️ Estrutura do Banco de Dados
sql-- database.sql

-- Tabela de Empresas (Origem/Destino)
CREATE TABLE companies (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    cnpj VARCHAR(18) UNIQUE NOT NULL,
    email VARCHAR(255) NOT NULL,
    type ENUM('origin', 'destination', 'both') DEFAULT 'both',
    api_key VARCHAR(255) UNIQUE,
    status ENUM('active', 'inactive', 'suspended') DEFAULT 'active',
    monthly_fee DECIMAL(10,2) DEFAULT 0.00,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_cnpj (cnpj),
    INDEX idx_api_key (api_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Tabela de Usuários
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    company_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    password VARCHAR(255) NOT NULL,
    role ENUM('admin', 'operator', 'validator') DEFAULT 'operator',
    status ENUM('active', 'inactive') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (company_id) REFERENCES companies(id) ON DELETE CASCADE,
    INDEX idx_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Tabela de Bilhetes
CREATE TABLE tickets (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ticket_code VARCHAR(100) UNIQUE NOT NULL,
    qr_code TEXT,
    
    -- Origem
    origin_company_id INT NOT NULL,
    origin_user_id INT NOT NULL,
    
    -- Destino
    destination_company_id INT NOT NULL,
    destination_address TEXT,
    
    -- Informações da Carga
    box_count INT NOT NULL DEFAULT 1,
    total_weight DECIMAL(10,2) NOT NULL, -- em KG
    total_volume DECIMAL(10,2) NOT NULL, -- em m³
    
    -- Dimensões (opcional, para cálculo detalhado)
    dimensions JSON, -- [{width, height, depth, weight}]
    
    -- Status e Controle
    status ENUM('pending', 'in_transit', 'validated', 'cancelled') DEFAULT 'pending',
    
    -- Valores
    transport_value DECIMAL(10,2) DEFAULT 0.00,
    saas_fee DECIMAL(10,2) DEFAULT 0.00,
    total_value DECIMAL(10,2) DEFAULT 0.00,
    
    -- Datas
    issued_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    validated_at TIMESTAMP NULL,
    
    -- Metadados
    metadata JSON,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (origin_company_id) REFERENCES companies(id),
    FOREIGN KEY (destination_company_id) REFERENCES companies(id),
    FOREIGN KEY (origin_user_id) REFERENCES users(id),
    
    INDEX idx_ticket_code (ticket_code),
    INDEX idx_status (status),
    INDEX idx_origin (origin_company_id),
    INDEX idx_destination (destination_company_id),
    INDEX idx_issued_at (issued_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Tabela de Validações
CREATE TABLE validations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ticket_id INT NOT NULL,
    validator_user_id INT NOT NULL,
    
    -- Dados da Validação
    validated_weight DECIMAL(10,2), -- peso real conferido
    validated_volume DECIMAL(10,2), -- volume real conferido
    validated_box_count INT,
    
    -- Diferenças
    weight_difference DECIMAL(10,2),
    volume_difference DECIMAL(10,2),
    
    -- Observações
    notes TEXT,
    photos JSON, -- URLs de fotos da carga
    
    -- Localização
    latitude DECIMAL(10,8),
    longitude DECIMAL(11,8),
    
    validated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (ticket_id) REFERENCES tickets(id) ON DELETE CASCADE,
    FOREIGN KEY (validator_user_id) REFERENCES users(id),
    
    INDEX idx_ticket (ticket_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Tabela de Precificação
CREATE TABLE pricing_rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    
    -- Regras de cálculo
    base_price DECIMAL(10,2) DEFAULT 0.00,
    price_per_kg DECIMAL(10,2) DEFAULT 0.00,
    price_per_m3 DECIMAL(10,2) DEFAULT 0.00,
    price_per_box DECIMAL(10,2) DEFAULT 0.00,
    
    -- Taxa SaaS (%)
    saas_percentage DECIMAL(5,2) DEFAULT 5.00,
    
    -- Faixas de peso/volume com desconto
    weight_tiers JSON, -- [{min, max, discount_percentage}]
    volume_tiers JSON,
    
    -- Vigência
    valid_from DATE,
    valid_until DATE,
    
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Tabela de Transações Financeiras
CREATE TABLE transactions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ticket_id INT NOT NULL,
    
    -- Valores
    transport_amount DECIMAL(10,2) NOT NULL,
    saas_amount DECIMAL(10,2) NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    
    -- Responsáveis
    company_id INT NOT NULL, -- empresa que será cobrada
    
    -- Controle de pagamento
    status ENUM('pending', 'paid', 'cancelled', 'refunded') DEFAULT 'pending',
    payment_method VARCHAR(50),
    payment_reference VARCHAR(255),
    
    paid_at TIMESTAMP NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (ticket_id) REFERENCES tickets(id),
    FOREIGN KEY (company_id) REFERENCES companies(id),
    
    INDEX idx_company (company_id),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Tabela de Logs/Auditoria
CREATE TABLE audit_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    entity_type VARCHAR(50), -- ticket, validation, transaction
    entity_id INT,
    action VARCHAR(50), -- create, update, delete, validate
    old_values JSON,
    new_values JSON,
    ip_address VARCHAR(45),
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL,
    INDEX idx_entity (entity_type, entity_id),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Inserir regra de precificação padrão
INSERT INTO pricing_rules (name, base_price, price_per_kg, price_per_m3, price_per_box, saas_percentage, valid_from, is_active) 
VALUES ('Padrão 2025', 10.00, 2.50, 15.00, 5.00, 5.00, '2025-01-01', TRUE);
🔧 Backend - CodeIgniter 4
1. app/Config/Routes.php
php<?php

namespace Config;

$routes = Services::routes();

// Auth
$routes->group('api/auth', function($routes) {
    $routes->post('login', 'API\AuthController::login');
    $routes->post('register', 'API\AuthController::register');
    $routes->get('me', 'API\AuthController::me', ['filter' => 'auth']);
    $routes->post('logout', 'API\AuthController::logout', ['filter' => 'auth']);
});

// Companies
$routes->group('api/companies', ['filter' => 'auth'], function($routes) {
    $routes->get('/', 'API\CompanyController::index');
    $routes->get('(:num)', 'API\CompanyController::show/$1');
    $routes->post('/', 'API\CompanyController::create');
    $routes->put('(:num)', 'API\CompanyController::update/$1');
});

// Tickets
$routes->group('api/tickets', ['filter' => 'auth'], function($routes) {
    $routes->get('/', 'API\TicketController::index');
    $routes->get('(:num)', 'API\TicketController::show/$1');
    $routes->post('/', 'API\TicketController::create');
    $routes->put('(:num)', 'API\TicketController::update/$1');
    $routes->delete('(:num)', 'API\TicketController::cancel/$1');
    $routes->get('code/(:segment)', 'API\TicketController::getByCode/$1');
});

// Validations
$routes->group('api/validations', ['filter' => 'auth'], function($routes) {
    $routes->post('/', 'API\ValidationController::validate');
    $routes->get('ticket/(:num)', 'API\ValidationController::getByTicket/$1');
});

// Billing
$routes->group('api/billing', ['filter' => 'auth'], function($routes) {
    $routes->get('dashboard', 'API\BillingController::dashboard');
    $routes->get('transactions', 'API\BillingController::transactions');
    $routes->get('report', 'API\BillingController::report');
});
2. app/Controllers/API/TicketController.php
php<?php

namespace App\Controllers\API;

use CodeIgniter\RESTful\ResourceController;
use App\Models\TicketModel;
use App\Models\CompanyModel;
use App\Libraries\PricingCalculator;

class TicketController extends ResourceController
{
    protected $modelName = 'App\Models\TicketModel';
    protected $format = 'json';

    public function index()
    {
        $user = $this->request->user;
        $companyId = $user['company_id'];
        
        $model = new TicketModel();
        
        $tickets = $model->where('origin_company_id', $companyId)
                        ->orWhere('destination_company_id', $companyId)
                        ->orderBy('created_at', 'DESC')
                        ->paginate(20);
        
        return $this->respond([
            'success' => true,
            'data' => $tickets,
            'pager' => $model->pager->getDetails()
        ]);
    }

    public function create()
    {
        $user = $this->request->user;
        $data = $this->request->getJSON(true);
        
        // Validação
        $validation = \Config\Services::validation();
        $validation->setRules([
            'destination_company_id' => 'required|integer',
            'box_count' => 'required|integer|greater_than[0]',
            'total_weight' => 'required|decimal|greater_than[0]',
            'total_volume' => 'required|decimal|greater_than[0]',
        ]);
        
        if (!$validation->run($data)) {
            return $this->fail($validation->getErrors(), 400);
        }
        
        // Gerar código do bilhete
        $ticketCode = $this->generateTicketCode();
        
        // Calcular valores
        $calculator = new PricingCalculator();
        $pricing = $calculator->calculate(
            $data['total_weight'],
            $data['total_volume'],
            $data['box_count']
        );
        
        // Preparar dados
        $ticketData = [
            'ticket_code' => $ticketCode,
            'origin_company_id' => $user['company_id'],
            'origin_user_id' => $user['id'],
            'destination_company_id' => $data['destination_company_id'],
            'destination_address' => $data['destination_address'] ?? null,
            'box_count' => $data['box_count'],
            'total_weight' => $data['total_weight'],
            'total_volume' => $data['total_volume'],
            'dimensions' => json_encode($data['dimensions'] ?? []),
            'transport_value' => $pricing['transport_value'],
            'saas_fee' => $pricing['saas_fee'],
            'total_value' => $pricing['total_value'],
            'status' => 'pending',
            'metadata' => json_encode($data['metadata'] ?? [])
        ];
        
        $model = new TicketModel();
        
        if ($model->insert($ticketData)) {
            $ticketId = $model->getInsertID();
            
            // Gerar QR Code
            $qrCode = $this->generateQRCode($ticketCode);
            $model->update($ticketId, ['qr_code' => $qrCode]);
            
            // Criar transação
            $this->createTransaction($ticketId, $pricing);
            
            $ticket = $model->find($ticketId);
            
            return $this->respondCreated([
                'success' => true,
                'message' => 'Bilhete criado com sucesso',
                'data' => $ticket
            ]);
        }
        
        return $this->fail('Erro ao criar bilhete', 500);
    }

    public function getByCode($code)
    {
        $model = new TicketModel();
        $ticket = $model->where('ticket_code', $code)->first();
        
        if (!$ticket) {
            return $this->failNotFound('Bilhete não encontrado');
        }
        
        return $this->respond([
            'success' => true,
            'data' => $ticket
        ]);
    }

    private function generateTicketCode()
    {
        return strtoupper(uniqid('TKT-') . '-' . bin2hex(random_bytes(4)));
    }

    private function generateQRCode($ticketCode)
    {
        // Implementar geração de QR Code
        // Pode usar biblioteca como chillerlan/php-qrcode
        return base64_encode("QR-{$ticketCode}");
    }

    private function createTransaction($ticketId, $pricing)
    {
        $transactionModel = model('TransactionModel');
        $ticketModel = new TicketModel();
        $ticket = $ticketModel->find($ticketId);
        
        $transactionModel->insert([
            'ticket_id' => $ticketId,
            'transport_amount' => $pricing['transport_value'],
            'saas_amount' => $pricing['saas_fee'],
            'total_amount' => $pricing['total_value'],
            'company_id' => $ticket['origin_company_id'],
            'status' => 'pending'
        ]);
    }
}
3. app/Controllers/API/ValidationController.php
php<?php

namespace App\Controllers\API;

use CodeIgniter\RESTful\ResourceController;
use App\Models\TicketModel;
use App\Models\ValidationModel;

class ValidationController extends ResourceController
{
    protected $format = 'json';

    public function validate()
    {
        $user = $this->request->user;
        $data = $this->request->getJSON(true);
        
        // Validação
        $validation = \Config\Services::validation();
        $validation->setRules([
            'ticket_code' => 'required|string',
            'validated_weight' => 'required|decimal',
            'validated_volume' => 'required|decimal',
            'validated_box_count' => 'required|integer'
        ]);
        
        if (!$validation->run($data)) {
            return $this->fail($validation->getErrors(), 400);
        }
        
        // Buscar bilhete
        $ticketModel = new TicketModel();
        $ticket = $ticketModel->where('ticket_code', $data['ticket_code'])->first();
        
        if (!$ticket) {
            return $this->failNotFound('Bilhete não encontrado');
        }
        
        // Verificar se o destino corresponde à empresa do usuário
        if ($ticket['destination_company_id'] != $user['company_id']) {
            return $this->failUnauthorized('Este bilhete não pertence a sua empresa');
        }
        
        // Verificar se já foi validado
        if ($ticket['status'] == 'validated') {
            return $this->fail('Bilhete já foi validado', 400);
        }
        
        // Calcular diferenças
        $weightDiff = $data['validated_weight'] - $ticket['total_weight'];
        $volumeDiff = $data['validated_volume'] - $ticket['total_volume'];
        
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
            'notes' => $data['notes'] ?? null,
            'latitude' => $data['latitude'] ?? null,
            'longitude' => $data['longitude'] ?? null,
            'photos' => json_encode($data['photos'] ?? [])
        ];
        
        if ($validationModel->insert($validationData)) {
            // Atualizar status do bilhete
            $ticketModel->update($ticket['id'], [
                'status' => 'validated',
                'validated_at' => date('Y-m-d H:i:s')
            ]);
            
            // Atualizar transação para pago
            $transactionModel = model('TransactionModel');
            $transactionModel->where('ticket_id', $ticket['id'])
                           ->set(['status' => 'paid', 'paid_at' => date('Y-m-d H:i:s')])
                           ->update();
            
            return $this->respondCreated([
                'success' => true,
                'message' => 'Bilhete validado com sucesso',
                'data' => [
                    'validation' => $validationData,
                    'differences' => [
                        'weight' => $weightDiff,
                        'volume' => $volumeDiff
                    ]
                ]
            ]);
        }
        
        return $this->fail('Erro ao validar bilhete', 500);
    }

    public function getByTicket($ticketId)
    {
        $validationModel = new ValidationModel();
        $validations = $validationModel->where('ticket_id', $ticketId)->findAll();
        
        return $this->respond([
            'success' => true,
            'data' => $validations
        ]);
    }
}
4. app/Libraries/PricingCalculator.php
php<?php

namespace App\Libraries;

use App\Models\PricingModel;

class PricingCalculator
{
    public function calculate($weight, $volume, $boxCount)
    {
        $pricingModel = new PricingModel();
        $rule = $pricingModel->where('is_active', true)
                            ->orderBy('id', 'DESC')
                            ->first();
        
        if (!$rule) {
            throw new \Exception('Nenhuma regra de precificação ativa encontrada');
        }
        
        // Cálculo base
        $transportValue = $rule['base_price'];
        $transportValue += ($weight * $rule['price_per_kg']);
        $transportValue += ($volume * $rule['price_per_m3']);
        $transportValue += ($boxCount * $rule['price_per_box']);
        
        // Aplicar descontos por faixa (se houver)
        $transportValue = $this->applyTierDiscounts($transportValue, $weight, $volume, $rule);
        
        // Calcular taxa SaaS
        $saasFee = $transportValue * ($rule['saas_percentage'] / 100);
        
        // Total
        $totalValue = $transportValue + $saasFee;
        
        return [
            'transport_value' => round($transportValue, 2),
            'saas_fee' => round($saasFee, 2),
            'total_value' => round($totalValue, 2),
            'rule_applied' => $rule['name']
        ];
    }
    
    private function applyTierDiscounts($baseValue, $weight, $volume, $rule)
    {
        $discount = 0;
        
        // Descontos por peso
        if (!empty($rule['weight_tiers'])) {
            $weightTiers = json_decode($rule['weight_tiers'], true);
            foreach ($weightTiers as $tier) {
                if ($weight >= $tier['min'] && $weight <= $tier['max']) {
                    $discount += ($baseValue * $tier['discount_percentage'] / 100);
                    break;
                }
            }
        }
        
        return $baseValue - $discount;
    }
}
⚛️ Frontend - React
1. src/services/api.js
javascriptimport axios from 'axios';

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
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export const authAPI = {
  login: (credentials) => api.post('/auth/login', credentials),
  register: (data) => api.post('/auth/register', data),
  me: () => api.get('/auth/me'),
  logout: () => api.post('/auth/logout'),
};

export const ticketAPI = {
  getAll: (params) => api.get('/tickets', { params }),
  getById: (id) => api.get(`/tickets/${id}`),
  getByCode: (code) => api.get(`/tickets/code/${code}`),
  create: (data) => api.post('/tickets', data),
  update: (id, data) => api.put(`/tickets/${id}`, data),
  cancel: (id) => api.delete(`/tickets/${id}`),
};

export const validationAPI = {
  validate: (data) => api.post('/validations', data),
  getByTicket: (ticketId) => api.get(`/validations/ticket/${ticketId}`),
};

export const billingAPI = {
  getDashboard: () => api.get('/billing/dashboard'),
  getTransactions: (params) => api.get('/billing/transactions', { params }),
  getReport: (params) => api.get('/billing/report', { params }),
};

export default api;
2. src/components/Tickets/CreateTicket.jsx
javascriptimport React, { useState, useEffect } from 'react';
import { ticketAPI } from '../../services/api';

const CreateTicket = () => {
  const [formData, setFormData] = useState({
    destination_company_id: '',
    destination_address: '',
    box_count: 1,
    total_weight: '',
    total_volume: '',
    dimensions: [],
  });

  const [boxes, setBoxes] = useState([{
    width: '',
    height: '',
    depth: '',
    weight: ''
  }]);

  const [pricing, setPricing] = useState(null);
  const [loading, setLoading] = useState(false);

  const handleBoxChange = (index, field, value) => {
    const newBoxes = [...boxes];
    newBoxes[index][field] = value;
    setBoxes(newBoxes);
    calculateTotals(newBoxes);
  };

  const addBox = () => {
    setBoxes([...boxes, { width: '', height: '', depth: '', weight: '' }]);
  };

  const removeBox = (index) => {
    const newBoxes = boxes.filter((_, i) => i !== index);
    setBoxes(newBoxes);
    calculateTotals(newBoxes);
  };

  const calculateTotals = (boxList) => {
    let totalWeight = 0;
    let totalVolume = 0;

    boxList.forEach(box => {
      if (box.weight) totalWeight += parseFloat(box.weight);
      if (box.width && box.height && box.depth) {
        const volume = (parseFloat(box.width) * parseFloat(box.height) * parseFloat(box.depth)) / 1000000; // cm³ para m³
        totalVolume += volume;
      }
    });

    setFormData(prev => ({
      ...prev,
      total_weight: totalWeight.toFixed(2),
      total_volume: totalVolume.toFixed(4),
      box_count: boxList.length,
      dimensions: boxList
    }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);

    try {
      const response = await ticketAPI.create(formData);
      
      if (response.data.success) {
        alert('Bilhete criado com sucesso!');
        // Redirecionar ou limpar formulário
        console.log('Ticket:', response.data.data);
      }
    } catch (error) {
      alert('Erro ao criar bilhete: ' + (error.response?.data?.message || error.message));
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="max-w-4xl mx-auto p-6 bg-white rounded-lg shadow">
      <h2 className="text-2xl font-bold mb-6">Emitir Novo Bilhete</h2>
      
      <form onSubmit={handleSubmit} className="space-y-6">
        {/* Informações de Destino */}
        <div className="border-b pb-4">
          <h3 className="text-lg font-semibold mb-4">Destino</h3>
          
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
              <label className="block text-sm font-medium mb-2">Empresa Destino</label>
              <select
                className="w-full p-2 border rounded"
                value={formData.destination_company_id}
                onChange={(e) => setFormData({...formData, destination_company_id: e.target.value})}
                required
              >
                <option value="">Selecione...</option>
                {/* Carregar empresas da API */}
              </select>
            </div>

            <div>
              <label className="block text-sm font-medium mb-2">Endereço</label>
              <input
                type="text"
                className="w-full p-2 border rounded"
                value={formData.destination_address}
                onChange={(e) => setFormData({...formData, destination_address: e.target.value})}
                placeholder="Rua, número, bairro..."
              />
            </div>
          </div>
        </div>

        {/* Caixas */}
        <div className="border-b pb-4">
          <div className="flex justify-between items-center mb-4">
            <h3 className="text-lg font-semibold">Caixas ({boxes.length})</h3>
            <button
              type="button"
              onClick={addBox}
              className="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700"
            >
              + Adicionar Caixa
            </button>
          </div>

          {boxes.map((box, index) => (
            <div key={index} className="bg-gray-50 p-4 rounded mb-3">
              <div className="flex justify-between items-center mb-3">
                <span className="font-medium">Caixa {index + 1}</span>
                {boxes.length > 1 && (
                  <button
                    type="button"
                    onClick={() => removeBox(index)}
                    className="text-red-600 hover:text-red-800"
                  >
                    Remover
                  </button>
                )}
              </div>

              <div className="grid grid-cols-2 md:grid-cols-4 gap-3">
                <div>
                  <label className="block text-xs mb-1">Largura (cm)</label>
                  <input
                    type="number"
                    step="0.01"
                    className="w-full p-2 border rounded text-sm"
                    value={box.width}
                    onChange={(e) => handleBoxChange(index, 'width', e.target.value)}
                    required
                  />
                </div>
                <div>
                  <label className="block text-xs mb-1">Altura (cm)</label>
                  <input
                    type="number"
                    step="0.01"
                    className="w-full p-2 border rounded text-sm"
                    value={box.height}
                    onChange={(e) => handleBoxChange(index, 'height', e.target.value)}
                    required
                  />
                </div>
                <div>
                  <label className="block text-xs mb-1">Profundidade (cm)</label>
                  <input
                    type="number"
                    step="0.01"
                    className="w-full p-2 border rounded text-sm"
                    value={box.depth}
                    onChange={(e) => handleBoxChange(index, 'depth', e.target.value)}
                    required
                  />
                </div>
                <div>
                  <label className="block text-xs mb-1">Peso (kg)</label>
                  <input
                    type="number"
                    step="0.01"
                    className="w-full p-2 border rounded text-sm"
                    value={box.weight}
                    onChange={(e) => handleBoxChange(index, 'weight', e.target.value)}
                    required
                  />
                </div>
              </div>
            </div>
          ))}
        </div>

        {/* Totais */}
        <div className="bg-blue-50 p-4 rounded">
          <h3 className="text-lg font-semibold mb-3">Resumo</h3>
          <div className="grid grid-cols-3 gap-4">
            <div>
              <p className="text-sm text-gray-600">Total de Caixas</p>
              <p className="text-xl font-bold">{formData.box_count}</p>
            </div>
            <div>
              <p className="text-sm text-gray-600">Peso Total</p>
              <p className="text-xl font-bold">{formData.total_weight || '0.00'} kg</p>
            </div>
            <div>
              <p className="text-sm text-gray-600">Volume Total</p>
              <p className="text-xl font-bold">{formData.total_volume || '0.00'} m³</p>
            </div>
          </div>
        </div>

        {/* Botões */}
        <div className="flex justify-end space-x-3">
          <button
            type="button"
            className="px-6 py-2 border rounded hover:bg-gray-50"
            onClick={() => window.history.back()}
          >
            Cancelar
          </button>
          <button
            type="submit"
            disabled={loading}
            className="px-6 py-2 bg-green-600 text-white rounded hover:bg-green-700 disabled:bg-gray-400"
          >
            {loading ? 'Criando...' : 'Emitir Bilhete'}
          </button>
        </div>
      </form>
    </div>
  );
};

export default CreateTicket;
3. src/components/Validation/ValidateTicket.jsx
javascriptimport React, { useState } from 'react';
import { validationAPI, ticketAPI } from '../../services/api';
import { QrReader } from 'react-qr-reader';

const ValidateTicket = () => {
  const [ticketCode, setTicketCode] = useState('');
  const [ticket, setTicket] = useState(null);
  const [showScanner, setShowScanner] = useState(false);
  const [validationData, setValidationData] = useState({
    validated_weight: '',
    validated_volume: '',
    validated_box_count: '',
    notes: '',
    photos: []
  });

  const searchTicket = async (code) => {
    try {
      const response = await ticketAPI.getByCode(code);
      if (response.data.success) {
        setTicket(response.data.data);
        setValidationData(prev => ({
          ...prev,
          validated_box_count: response.data.data.box_count
        }));
      }
    } catch (error) {
      alert('Bilhete não encontrado');
    }
  };

  const handleScan = (result) => {
    if (result) {
      setTicketCode(result.text);
      searchTicket(result.text);
      setShowScanner(false);
    }
  };

  const handleValidate = async (e) => {
    e.preventDefault();

    try {
      const response = await validationAPI.validate({
        ticket_code: ticketCode,
        ...validationData
      });

      if (response.data.success) {
        alert('Bilhete validado com sucesso!');
        
        // Mostrar diferenças se houver
        const { differences } = response.data.data;
        if (Math.abs(differences.weight) > 0.5 || Math.abs(differences.volume) > 0.01) {
          alert(`Atenção! Diferenças encontradas:\nPeso: ${differences.weight.toFixed(2)} kg\nVolume: ${differences.volume.toFixed(4)} m³`);
        }
        
        // Resetar
        setTicket(null);
        setTicketCode('');
        setValidationData({
          validated_weight: '',
          validated_volume: '',
          validated_box_count: '',
          notes: ''
        });
      }
    } catch (error) {
      alert('Erro ao validar: ' + (error.response?.data?.message || error.message));
    }
  };

  return (
    <div className="max-w-3xl mx-auto p-6">
      <h2 className="text-2xl font-bold mb-6">Validar Bilhete</h2>

      {/* Scanner / Busca */}
      <div className="bg-white rounded-lg shadow p-6 mb-6">
        <div className="flex space-x-3 mb-4">
          <input
            type="text"
            className="flex-1 p-3 border rounded"
            placeholder="Digite o código do bilhete..."
            value={ticketCode}
            onChange={(e) => setTicketCode(e.target.value)}
            onKeyPress={(e) => e.key === 'Enter' && searchTicket(ticketCode)}
          />
          <button
            onClick={() => searchTicket(ticketCode)}
            className="px-6 py-3 bg-blue-600 text-white rounded hover:bg-blue-700"
          >
            Buscar
          </button>
          <button
            onClick={() => setShowScanner(!showScanner)}
            className="px-6 py-3 bg-purple-600 text-white rounded hover:bg-purple-700"
          >
            📷 QR Code
          </button>
        </div>

        {showScanner && (
          <div className="mt-4">
            <QrReader
              constraints={{ facingMode: 'environment' }}
              onResult={handleScan}
              style={{ width: '100%' }}
            />
          </div>
        )}
      </div>

      {/* Informações do Bilhete */}
      {ticket && (
        <div className="bg-white rounded-lg shadow p-6 mb-6">
          <h3 className="text-xl font-bold mb-4">Informações do Bilhete</h3>
          
          <div className="grid grid-cols-2 gap-4 mb-4">
            <div>
              <p className="text-sm text-gray-600">Código</p>
              <p className="font-semibold">{ticket.ticket_code}</p>
            </div>
            <div>
              <p className="text-sm text-gray-600">Status</p>
              <span className={`px-3 py-1 rounded text-sm ${
                ticket.status === 'pending' ? 'bg-yellow-100 text-yellow-800' :
                ticket.status === 'validated' ? 'bg-green-100 text-green-800' :
                'bg-gray-100 text-gray-800'
              }`}>
                {ticket.status}
              </span>
            </div>
          </div>

          <div className="grid grid-cols-3 gap-4 p-4 bg-gray-50 rounded">
            <div>
              <p className="text-sm text-gray-600">Caixas</p>
              <p className="text-lg font-bold">{ticket.box_count}</p>
            </div>
            <div>
              <p className="text-sm text-gray-600">Peso Declarado</p>
              <p className="text-lg font-bold">{ticket.total_weight} kg</p>
            </div>
            <div>
              <p className="text-sm text-gray-600">Volume Declarado</p>
              <p className="text-lg font-bold">{ticket.total_volume} m³</p>
            </div>
          </div>
        </div>
      )}

      {/* Formulário de Validação */}
      {ticket && ticket.status === 'pending' && (
        <form onSubmit={handleValidate} className="bg-white rounded-lg shadow p-6">
          <h3 className="text-xl font-bold mb-4">Conferência da Carga</h3>

          <div className="grid grid-cols-3 gap-4 mb-4">
            <div>
              <label className="block text-sm font-medium mb-2">Quantidade de Caixas</label>
              <input
                type="number"
                className="w-full p-3 border rounded"
                value={validationData.validated_box_count}
                onChange={(e) => setValidationData({...validationData, validated_box_count: e.target.value})}
                required
              />
            </div>
            <div>
              <label className="block text-sm font-medium mb-2">Peso Real (kg)</label>
              <input
                type="number"
                step="0.01"
                className="w-full p-3 border rounded"
                value={validationData.validated_weight}
                onChange={(e) => setValidationData({...validationData, validated_weight: e.target.value})}
                required
              />
            </div>
            <div>
              <label className="block text-sm font-medium mb-2">Volume Real (m³)</label>
              <input
                type="number"
                step="0.0001"
                className="w-full p-3 border rounded"
                value={validationData.validated_volume}
                onChange={(e) => setValidationData({...validationData, validated_volume: e.target.value})}
                required
              />
            </div>
          </div>

          <div className="mb-4">
            <label className="block text-sm font-medium mb-2">Observações</label>
            <textarea
              className="w-full p-3 border rounded"
              rows="3"
              value={validationData.notes}
              onChange={(e) => setValidationData({...validationData, notes: e.target.value})}
              placeholder="Adicione observações sobre a conferência..."
            />
          </div>

          <button
            type="submit"
            className="w-full py-3 bg-green-600 text-white rounded-lg hover:bg-green-700 font-semibold"
          >
            ✓ Validar Bilhete
          </button>
        </form>
      )}
    </div>
  );
};

export default ValidateTicket;
🐳 Docker Setup
yaml# docker-compose.yml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: logistics_ticketing
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./database.sql:/docker-entrypoint-initdb.d/init.sql

  backend:
    build: ./backend
    ports:
      - "8080:8080"
    volumes:
      - ./backend:/var/www/html
    depends_on:
      - mysql
    environment:
      DB_HOST: mysql
      DB_DATABASE: logistics_ticketing
      DB_USERNAME: root
      DB_PASSWORD: root

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      REACT_APP_API_URL: http://localhost:8080/api

volumes:
  mysql_data:
📊 Fluxo do Sistema

Emissão do Bilhete (Origem)

Usuário preenche dados da carga
Sistema calcula preço automaticamente
Gera código único e QR Code
Cria transação pendente


Validação (Destino)

Scanner QR Code ou busca por código
Confere peso/volume real
Registra diferenças
Marca transação como paga


Faturamento

Dashboard com métricas
Relatórios de transações
Cálculo automático de taxas



Este é um sistema completo e escalável! Quer que eu detalhe alguma parte específica?