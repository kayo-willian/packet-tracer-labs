# Lab, DNS Resolution with a Web Server

## Objective
Set up a DNS server so a PC can reach a website by typing its domain name
instead of its raw IP address, and see what happens when a PC tries to
reach that name before DNS is configured on it.

## Topology
![topology](images/topology.png)

```
PC0, PC1 -- Switch0 -- Router0 -- Switch1 -- Server0 (DNS + Web)
```

- PC0 and PC1 sit on one network, connected through Switch0 to Router0.
- Server0 sits on a separate network, connected through Switch1 to
  Router0. It runs both the DNS service and the HTTP service.
- Static IP addressing was used on every device, no DHCP this time, so the
  DNS Server field on each PC could be set manually.

## Addressing table

| Device | IP | Mask | Gateway | DNS Server |
|--------|----|----|---------|------------|
| Router0, Gi0/0 | `192.168.1.1` | `255.255.255.0` | n/a | n/a |
| Router0, Gi0/1 | `192.168.2.1` | `255.255.255.0` | n/a | n/a |
| PC0 | `192.168.1.2` | `255.255.255.0` | `192.168.1.1` | `192.168.2.10` |
| PC1 | `192.168.1.3` | `255.255.255.0` | `192.168.1.1` | `192.168.2.10` |
| Server0 | `192.168.2.10` | `255.255.255.0` | `192.168.2.1` | n/a |

## DNS configuration
DNS service enabled on Server0, with a single A record:

![dns config](images/dns-config.png)

| Name | Type | Address |
|------|------|---------|
| `www.empresa.com` | A Record | `192.168.2.10` |

## The web page
A simple HTML page was placed on the server to have something real to load
once DNS resolution worked. Full source below.

```html
<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Empresa Ltda.</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: Arial, sans-serif;
            background-color: #f4f4f4;
            color: #333;
            line-height: 1.6;
        }

        header {
            background-color: #6d4aff;
            color: white;
            padding: 2rem;
            text-align: center;
        }

        nav {
            background-color: #5a3fd6;
            padding: 1rem;
            text-align: center;
        }

        nav a {
            color: white;
            text-decoration: none;
            margin: 0 15px;
            font-weight: bold;
        }

        nav a:hover {
            text-decoration: underline;
        }

        main {
            max-width: 900px;
            margin: 2rem auto;
            padding: 2rem;
            background-color: white;
            border-radius: 8px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
        }

        section {
            margin-bottom: 2rem;
        }

        h1 {
            margin-bottom: 1rem;
        }

        h2 {
            color: #6d4aff;
            margin-bottom: 1rem;
        }

        p {
            margin-bottom: 1rem;
        }

        footer {
            background-color: #333;
            color: white;
            text-align: center;
            padding: 1rem;
            margin-top: 2rem;
        }

        .contact-info {
            background-color: #e8e4ff;
            padding: 1rem;
            border-left: 4px solid #6d4aff;
        }
    </style>
</head>
<body>
    <header>
        <h1>Bem-vindo à Empresa Ltda.</h1>
        <p>Soluções inovadoras para o seu negócio</p>
    </header>

    <nav>
        <a href="#home">Início</a>
        <a href="#sobre">Sobre</a>
        <a href="#servicos">Serviços</a>
        <a href="#contato">Contato</a>
    </nav>

    <main id="home">
        <section>
            <h2>Sobre Nós</h2>
            <p>Fundada em 2020, a Empresa Ltda. se dedica a oferecer as melhores soluções tecnológicas para empresas de todos os tamanhos.</p>
            <p>Nossa missão é transformar negócios através da inovação e excelência no atendimento.</p>
        </section>

        <section id="servicos">
            <h2>Nossos Serviços</h2>
            <ul>
                <li>Consultoria em TI</li>
                <li>Desenvolvimento de Software</li>
                <li>Infraestrutura de Rede</li>
                <li>Suporte Técnico 24/7</li>
                <li>Cibersegurança</li>
            </ul>
        </section>

        <section id="contato">
            <h2>Entre em Contato</h2>
            <div class="contact-info">
                <p><strong>Email:</strong> contato@empresa.com</p>
                <p><strong>Telefone:</strong> (11) 9999-9999</p>
                <p><strong>Endereço:</strong> Av. Principal, 1000 - São Paulo, SP</p>
                <p><strong>Horário:</strong> Seg-Sex: 08:00 às 18:00</p>
            </div>
        </section>
    </main>

    <footer>
        <p>&copy; 2024 Empresa Ltda. Todos os direitos reservados.</p>
        <p>Página hospedada via DNS - www.empresa.com</p>
    </footer>
</body>
</html>
```

## Before and after
Before setting the DNS Server field on the PC, typing the domain name
fails, the PC has no way to translate it into an IP:

![before, host name unresolved](images/before.png)

After the DNS Server field was set to `192.168.2.10`, the same URL
resolves correctly and loads the page:

![after, page loaded](images/after.png)

## Problem found
While reviewing the PC configuration screenshots for this README, one of
them (PC1) shows `198.168.1.1` in the Default Gateway field, with a typo
in the first octet, it should read `192.168.1.1` to match the rest of the
network. This did not come up during testing since PC1 was not the one
used to test DNS resolution, but it is worth fixing before relying on that
PC for anything that needs to leave its own network, since a wrong gateway
address in that field would mean it never reaches devices outside its own
subnet.

## What I learned
- DNS exists to translate a human friendly name into an IP address, a PC
  cannot resolve a domain name on its own, it needs a DNS server it knows
  how to reach, configured in its own IP settings.
- The DNS Server field on a PC is not optional decoration, without it,
  typing a domain name simply fails, even if the underlying network and
  the web server are working perfectly.
- An A record is the basic building block of DNS, mapping one name to one
  IP address.
- The same device can run more than one service at once, in this case the
  same server handled both DNS and HTTP.
- Double checking every IP field carefully matters, a single typo in a
  gateway address is easy to miss and can silently break connectivity for
  that device later.

## Next step
Fix the gateway typo on PC1 and confirm it can resolve and reach the site
too, then try adding a second A record for a different name pointing to
the same or a different server.
