#!/usr/bin/env python3
# ecotrip.py

"""
Calculadora EcoTrip - Simulador de Impacto Ambiental para Viagens

Modo interativo:
    python ecotrip.py

Modo via linha de comando:
    python ecotrip.py --distancia 1000 --modo aviao_longa --passageiros 2 --ida-volta --noites 3 --hotel hotel_padrao

Opções extras:
    --json                → imprime resultado em JSON
    --no-interativo       → não usa modo interativo; exige argumentos
"""

from __future__ import annotations
import argparse
import json
from typing import Optional, Tuple

# ---------------------------------------------------------
# Fatores de emissão (kg CO2 por passageiro-km)
# ---------------------------------------------------------
FATORES_TRANSPORTE = {
    "carro": 0.192,
    "onibus": 0.080,
    "trem": 0.041,
    "aviao_curta": 0.255,
    "aviao_longa": 0.195
}

# Emissões por noite (kg CO2)
FATORES_HOTEL = {
    "hostel": 5.0,
    "hotel_padrao": 15.0,
    "hotel_luxo": 50.0
}

# ---------------------------------------------------------
# Funções utilitárias
# ---------------------------------------------------------
def parse_yesno(valor: Optional[str]) -> bool:
    """Converte entrada textual em booleano."""
    if valor is None:
        return False
    return str(valor).strip().lower() in ("s", "sim", "y", "yes", "1", "true", "t")


def calcula_emissao_transporte(dist_km: float, modo: str, passageiros: int = 1,
                               ida_e_volta: bool = True) -> Tuple[float, float]:
    modo = modo.lower()
    if modo not in FATORES_TRANSPORTE:
        raise ValueError(f"Modo desconhecido: {modo}. Opções: {', '.join(FATORES_TRANSPORTE.keys())}")

    fator = FATORES_TRANSPORTE[modo]
    multiplicador = 2 if ida_e_volta else 1

    total = dist_km * fator * multiplicador
    por_pessoa = total / max(1, passageiros)

    return total, por_pessoa


def calcula_emissao_hotel(noites: int, tipo: Optional[str]) -> float:
    if not tipo or noites <= 0:
        return 0.0

    tipo = tipo.lower()
    if tipo not in FATORES_HOTEL:
        raise ValueError(f"Tipo de hospedagem desconhecido: {tipo}. Opções: {', '.join(FATORES_HOTEL.keys())}")

    return noites * FATORES_HOTEL[tipo]


def formatar_saida_texto(dist_km: float, modo: str, passageiros: int, ida_e_volta: bool,
                         noites: int, hotel: Optional[str], transp_total: float,
                         transp_pessoa: float, hotel_total: float, total: float,
                         total_pessoa: float) -> str:

    linhas = []
    linhas.append("=== Resultado EcoTrip ===")
    linhas.append(f"Distância: {dist_km:.1f} km")
    linhas.append(f"Modo de transporte: {modo}")
    linhas.append(f"Ida e volta: {'Sim' if ida_e_volta else 'Não'}")
    linhas.append(f"Passageiros: {passageiros}")
    linhas.append("")
    linhas.append("--- Emissões (kg CO₂) ---")
    linhas.append(f"Transporte (total): {transp_total:.2f}")
    linhas.append(f"Transporte (por pessoa): {transp_pessoa:.2f}")
    linhas.append(f"Hospedagem (total): {hotel_total:.2f}")
    linhas.append("")
    linhas.append(f"Total estimado: {total:.2f} kg CO₂")
    linhas.append(f"Por pessoa: {total_pessoa:.2f} kg CO₂")
    linhas.append("")
    linhas.append("Dicas rápidas:")
    linhas.append("- Prefira trem ou ônibus quando possível.")
    linhas.append("- Compartilhe carro (reduz emissão por pessoa).")
    linhas.append("- Escolha hospedagens mais simples ou menos noites.")

    return "\n".join(linhas)

# ---------------------------------------------------------
# Modo principal
# ---------------------------------------------------------
def main():
    parser = argparse.ArgumentParser(description="Calculadora EcoTrip - estimativa de CO₂ para viagens")

    parser.add_argument("--distancia", type=float, help="Distância em km")
    parser.add_argument("--modo", type=str, help=f"Modo de transporte ({', '.join(FATORES_TRANSPORTE.keys())})")
    parser.add_argument("--passageiros", type=int, default=1, help="Número de passageiros")
    parser.add_argument("--ida-volta", action="store_true", help="Indica ida e volta")
    parser.add_argument("--noites", type=int, default=0, help="Número de noites")
    parser.add_argument("--hotel", type=str, help=f"Tipo de hospedagem ({', '.join(FATORES_HOTEL.keys())})")
    parser.add_argument("--json", action="store_true", help="Saída em JSON")
    parser.add_argument("--no-interativo", action="store_true", help="Força modo não interativo")

    args = parser.parse_args()

    # -----------------------------------------------------
    # Se faltar informação e NÃO usar modo obrigatório,
    # entra no modo interativo
    # -----------------------------------------------------
    usar_interativo = (
        not args.no_interativo and
        (args.distancia is None or args.modo is None)
    )

    if usar_interativo:
        print("=== EcoTrip (modo interativo) ===")

        dist = input("Distância (km): ").strip().replace(",", ".")
        distancia = float(dist) if dist else 0.0

        print("Modos disponíveis:", ", ".join(FATORES_TRANSPORTE.keys()))
        modo = input("Modo de transporte: ").strip().lower() or "carro"

        passageiros = int(input("Passageiros (padrão 1): ") or "1")
        ida_volta = parse_yesno(input("É ida e volta? (s/N): "))

        noites = int(input("Noites de hospedagem (0 para nenhum): ") or "0")
        print("Tipos de hospedagem:", ", ".join(FATORES_HOTEL.keys()))
        hotel = input("Tipo de hospedagem (opcional): ").strip().lower() or None

    else:
        if args.distancia is None or args.modo is None:
            print("Erro: use --no-interativo OU informe --distancia e --modo.")
            return

        distancia = args.distancia
        modo = args.modo
        passageiros = args.passageiros
        ida_volta = args.ida_volta
        noites = args.noites
        hotel = args.hotel

    # -----------------------------------------------------
    # Cálculos
    # -----------------------------------------------------
    transp_total, transp_pessoa = calcula_emissao_transporte(distancia, modo, passageiros, ida_volta)
    hotel_total = calcula_emissao_hotel(noites, hotel)

    total = transp_total + hotel_total
    total_pessoa = total / max(1, passageiros)

    # -----------------------------------------------------
    # Saída em JSON
    # -----------------------------------------------------
    if args.json:
        print(json.dumps({
            "distancia_km": distancia,
            "modo": modo,
            "passageiros": passageiros,
            "ida_volta": ida_volta,
            "noites": noites,
            "hotel": hotel,
            "transporte_total": transp_total,
            "transporte_por_pessoa": transp_pessoa,
            "hotel_total": hotel_total,
            "total_kg": total,
            "total_por_pessoa": total_pessoa
        }, indent=4, ensure_ascii=False))
        return

    # -----------------------------------------------------
    # Saída em texto
    # -----------------------------------------------------
    print(
        formatar_saida_texto(
            distancia, modo, passageiros, ida_volta,
            noites, hotel, transp_total, transp_pessoa,
            hotel_total, total, total_pessoa
        )
    )


if __name__ == "__main__":
    main()
