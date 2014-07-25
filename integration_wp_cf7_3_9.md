# Integrações RD Station
## Wordpress e Contact Form 7 3.9

### Criar ou configurar formulários do Contact Form 7 3.9

Quando for criar um formulário no painel do plugin, eu editar algum, é preciso padronizar os mesmos tipos de input para usarem sempre os mesmos nomes (por ex.: 'email' para email, 'nome' para nome, etc), assim seu formulário ficará mais ou menos assim:

```HTML
<p>Seu email (obrigatório) [email* email]</p>
<p>Seu nome [text nome]</p>
```

É obrigatório a presença de um campo <strong>email</strong> ou <strong>email_lead</strong>.

> É possível utilizar uma lista de outros campos já cadastrados na ferramenta de CRM do RD Station. Segue uma breve lista de opções:<ul><li>nome</li><li>telefone</li><li>empresa</li><li>cargo</li><li>twitter</li></ul>

É preciso também incluir dois novos campos para passar o identificador do formulário/página e o cookie utmz do Google Analytics (*preenchido automaticamente com um código Javascript presente também no fragmento abaixo):

```HTML
<div style="display:none;">
[text identificador "pagina contato"]
[text c_utmz id:cookieutmz ""]
</div>
<script type="text/javascript">
function read_cookie(a){var b=a+"=";var c=document.cookie.split(";");for(var d=0;d<c.length;d++){var e=c[d];while(e.charAt(0)==" ")e=e.substring(1,e.length);if(e.indexOf(b)==0){return e.substring(b.length,e.length)}}return null}try{document.getElementById("cookieutmz").value=read_cookie("__utmz")}catch(err){}
</script>
```

### Editar tema

Para enviar os dados do formulário para o RD Station, insira o código abaixo no final do arquivo <code>functions.php</code> do seu tema do Wordpress.

Antes de salvar, é preciso alterar o código inserindo o token RD Station de sua conta (encontrado em https://www.rdstation.com.br/docs/api ).

```PHP
<?php
/**
 * RD Station - Integrações
 * addLeadConversionToRdstationCrm()
 * Envio de dados para a API de leads do RD Station
 *
 * Parâmetros:
 *     ($rdstation_token) - token da sua conta RD Station ( encontrado em https://www.rdstation.com.br/docs/api )
 *     ($identifier) - identificador da página ou evento ( por exemplo, 'pagina-contato' )
 *     ($data_array) - um Array com campos do formulário ( por exemplo, array('email' => 'teste@rdstation.com.br', 'name' =>'Fulano') )
 */
function addLeadConversionToRdstationCrm( $rdstation_token, $identifier, $data_array ) {
  $api_url = "http://www.rdstation.com.br/api/1.2/conversions";

  try {
    if (empty($data_array["token_rdstation"]) && !empty($rdstation_token)) { $data_array["token_rdstation"] = $rdstation_token; }
    if (empty($data_array["identificador"]) && !empty($identifier)) { $data_array["identificador"] = $identifier; }
    if (empty($data_array["email"])) { $data_array["email"] = $data_array["your-email"]; }
    if (empty($data_array["c_utmz"])) { $data_array["c_utmz"] = $_COOKIE["__utmz"]; }
    unset($data_array["password"], $data_array["password_confirmation"], $data_array["senha"], 
          $data_array["confirme_senha"], $data_array["captcha"], $data_array["_wpcf7"], 
          $data_array["_wpcf7_version"], $data_array["_wpcf7_unit_tag"], $data_array["_wpnonce"], 
          $data_array["_wpcf7_is_ajax_call"], $data_array["your-email"]);
      
    if ( !empty($data_array["token_rdstation"]) && !( empty($data_array["email"]) && empty($data_array["email_lead"]) ) ) {
      $data_query = http_build_query($data_array);
      if (in_array ('curl', get_loaded_extensions())) {
        $ch = curl_init($api_url);
        curl_setopt($ch, CURLOPT_POST, 1);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $data_query);
        curl_setopt($ch, CURLOPT_FOLLOWLOCATION, 1);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
        curl_exec($ch);
        curl_close($ch);
      } else {
        $params = array('http' => array('method' => 'POST', 'content' => $data_query, 'ignore_errors' => true));
        $ctx = stream_context_create($params); 
        $fp = @fopen($api_url, 'rb', false, $ctx);
      }
    }
  } catch (Exception $e) { }
}
function addLeadConversionToRdstationCrmViaWpCf7( $cf7 ) {
  $token_rdstation = "SEU_TOKEN_RDSTATION_AQUI";
  $submission = WPCF7_Submission::get_instance();

  if ( $submission ) {
      $form_data = $submission->get_posted_data();
    }
  addLeadConversionToRdstationCrm($token_rdstation, null, $form_data);
}
add_action('wpcf7_mail_sent', 'addLeadConversionToRdstationCrmViaWpCf7');
?>
```

> É possível também inserir outros parâmetros do POST para enviar ao RD Station

### Avisos de conversão por email

O RD Station pode lhe enviar um email quando uma nova conversão for realizada em seu site.
Para isso, basta colocar o seu email na configuração da página da API https://www.rdstation.com.br/docs/api

### Erros

A API pode retornar erro caso:
 - (401) seu token RD Station esteja errado ou inválido;
 - (400) não esteja recebendo um identificador;
 - (400) não esteja recebendo a informação <strong>email</strong> ou <strong>email_lead</strong> do formulário;
