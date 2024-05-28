# AtividadeIA

using System;
using System.IO;
using System.Net.Http;
using System.Text.Json;
using System.Text.RegularExpressions;
using System.Threading.Tasks;

class Program
{
    static async Task Main(string[] args)
    {
        Console.Write("Nome: ");
        string nome = Console.ReadLine();

        Console.Write("Salário Bruto: ");
        decimal salarioBruto = decimal.Parse(Console.ReadLine());

        Console.Write("Desconto do INSS: ");
        decimal descontoINSS = decimal.Parse(Console.ReadLine());

        Console.Write("Número de Dependentes: ");
        int numeroDependentes = int.Parse(Console.ReadLine());

        Console.Write("Valor Total de Descontos: ");
        decimal valorDescontos = decimal.Parse(Console.ReadLine());

        Console.Write("CPF: ");
        string cpf = Console.ReadLine();
        if (!ValidarCPF(cpf))
        {
            Console.WriteLine("CPF inválido.");
            return;
        }

        Console.Write("CEP: ");
        string cep = Console.ReadLine();
        string endereco = await ConsultarCEP(cep);
        if (endereco == null)
        {
            Console.WriteLine("CEP inválido.");
            return;
        }

        decimal irrf = CalcularIRRF(salarioBruto, descontoINSS, numeroDependentes, valorDescontos);
        decimal salarioLiquido = salarioBruto - descontoINSS - irrf;

        Console.WriteLine($"Salário Líquido: {salarioLiquido:C}");
        Console.WriteLine($"Endereço: {endereco}");

        SalvarDados(nome, salarioBruto, descontoINSS, numeroDependentes, valorDescontos, cpf, cep, endereco, salarioLiquido);

        Console.WriteLine("Dados salvos com sucesso.");
    }

    static bool ValidarCPF(string cpf)
    {
        cpf = cpf.Replace(".", "").Replace("-", "");
        if (cpf.Length != 11)
            return false;

        for (int j = 0; j < 10; j++)
            if (cpf == new string(j.ToString()[0], 11))
                return false;

        int[] multiplicador1 = new int[9] { 10, 9, 8, 7, 6, 5, 4, 3, 2 };
        int[] multiplicador2 = new int[10] { 11, 10, 9, 8, 7, 6, 5, 4, 3, 2 };
        string tempCpf = cpf.Substring(0, 9);
        int soma = 0;

        for (int i = 0; i < 9; i++)
            soma += int.Parse(tempCpf[i].ToString()) * multiplicador1[i];

        int resto = soma % 11;
        if (resto < 2)
            resto = 0;
        else
            resto = 11 - resto;

        string digito = resto.ToString();
        tempCpf = tempCpf + digito;
        soma = 0;
        for (int i = 0; i < 10; i++)
            soma += int.Parse(tempCpf[i].ToString()) * multiplicador2[i];

        resto = soma % 11;
        if (resto < 2)
            resto = 0;
        else
            resto = 11 - resto;

        digito = digito + resto.ToString();

        return cpf.EndsWith(digito);
    }

    static async Task<string> ConsultarCEP(string cep)
    {
        using (HttpClient client = new HttpClient())
        {
            try
            {
                HttpResponseMessage response = await client.GetAsync($"https://viacep.com.br/ws/{cep}/json/");
                response.EnsureSuccessStatusCode();
                string responseBody = await response.Content.ReadAsStringAsync();
                var data = JsonSerializer.Deserialize<Endereco>(responseBody);
                if (data.erro)
                    return null;

                return $"{data.logradouro}, {data.bairro}, {data.localidade} - {data.uf}";
            }
            catch
            {
                return null;
            }
        }
    }

    static decimal CalcularIRRF(decimal salarioBruto, decimal descontoINSS, int numeroDependentes, decimal valorDescontos)
    {
        decimal baseCalculo = salarioBruto - descontoINSS - (numeroDependentes * 189.59m) - valorDescontos;

        decimal irrf = 0;
        if (baseCalculo <= 1903.98m)
        {
            irrf = 0;
        }
        else if (baseCalculo <= 2826.65m)
        {
            irrf = (baseCalculo * 0.075m) - 142.80m;
        }
        else if (baseCalculo <= 3751.05m)
        {
            irrf = (baseCalculo * 0.15m) - 354.80m;
        }
        else if (baseCalculo <= 4664.68m)
        {
            irrf = (baseCalculo * 0.225m) - 636.13m;
        }
        else
        {
            irrf = (baseCalculo * 0.275m) - 869.36m;
        }

        return irrf < 0 ? 0 : irrf;
    }

    static void SalvarDados(string nome, decimal salarioBruto, decimal descontoINSS, int numeroDependentes, decimal valorDescontos, string cpf, string cep, string endereco, decimal salarioLiquido)
    {
        string filePath = "dados_trabalhadores.csv";
        string[] lines = File.Exists(filePath) ? File.ReadAllLines(filePath) : new string[0];

        using (StreamWriter writer = new StreamWriter(filePath))
        {
            bool exists = false;
            foreach (var line in lines)
            {
                string[] fields = line.Split(',');
                if (fields[0] == cpf)
                {
                    writer.WriteLine($"{cpf},{nome},{salarioBruto},{descontoINSS},{numeroDependentes},{valorDescontos},{cep},{endereco},{salarioLiquido}");
                    exists = true;
                }
                else
                {
                    writer.WriteLine(line);
                }
            }
            if (!exists)
            {
                writer.WriteLine($"{cpf},{nome},{salarioBruto},{descontoINSS},{numeroDependentes},{valorDescontos},{cep},{endereco},{salarioLiquido}");
            }
        }
    }
}

public class Endereco
{
    public string logradouro { get; set; }
    public string bairro { get; set; }
    public string localidade { get; set; }
    public string uf { get; set; }
    public bool erro { get; set; }
}

