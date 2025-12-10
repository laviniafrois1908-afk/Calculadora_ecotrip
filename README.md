# Calculadora_ecotrip
#!/usr/bin/env python3
# ecotrip.py
"""
Calculadora EcoTrip - Simulador de Impacto Ambiental para Viagens

Uso interativo:
    python ecotrip.py

Uso via linha de comando:
    python ecotrip.py --distance 1000 --mode aviao_longa --passengers 2 --roundtrip --nights 3 --hotel hotel_padrao

Opções:
    --json      Saída em JSON
    --no-interactive  Força modo não interativo (falha se parâmetros obrigatórios não fornecidos)
"""
from __future__ import annotations
import argparse
import json
from typing import Optional, Tuple

# Fatores de emissão aproximados (kg CO2 por passageiro-km)
EMISSION_FACTORS = {
    "carro": 0.192,
    "onibus": 0.080,
    "trem": 0.041,
    "aviao_curta": 0.255,
    "aviao_longa": 0.195
}

# Emissão por noite de hospedagem (kg CO2 por noite)
HOTEL_FACTORS = {
    "hostel": 5.0,
    "hotel_padrao": 15.0,
    "hotel_luxo": 50.0
}


def parse_yesno(s: Optional[str]) -> bool:
    if s is None:
        return False
    return str(s).strip().lower() in ("s", "sim", "y", "yes", "1", "true", "t")


def calcula_emissao_transporte(dist_km: float, modo: str, passageiros: int = 1, ida_e_volta: bool = True) -> Tuple[float, float]:
    """
    Retorna (total_kg, por_pessoa_kg)
    """
    modo_key = (modo or "carro").lower()
    fator = EMISSION_FACTORS.get(modo_key)
    if fator is None:
        raise ValueError(f"Modo de transporte desconhecido: '{modo}'. Opções: {', '.join(EMISSION_FACTORS.keys())}")
    multiplicador = 2 if ida_e_volta else 1
    total_kg = dist_km * fator * multiplicador
    per_person = total_kg / max(1, passageiros)
    return total_kg, per_person


def calcula_emissao_hotel(noites: int, tipo: Optional[str]) -> float:
    if noites <= 0 or not tipo:
        return 0.0
    tipo_key = tipo.lower()
    fator = HOTEL_FACTORS.get(tipo_key)
    if fator is None:
        raise ValueError(f"Tipo de hotel desconhecido: '{tipo}'. Opções: {', '.join(HOTEL_FACTORS.keys())}")
    return noites * fator


def formato_texto_saida(distance_km: float, modo: str, passageiros: int, ida_e_volta: bool, noites: int, hotel: Optional[str],
                        transporte_total: float, transporte_por_pessoa: float, hotel_total: float, total_kg: float, total_por_pessoa: float) -> str:
    lines = []
    lines.append("=== Resultado EcoTrip ===")
    lines.append(f"Distância (km): {distance_km:.1f}")
    lines.append(f"Modo: {modo}")
    lines.append(f"Ida e volta: {'Sim' if ida_e_volta else 'Não'}")
    lines.append(f"Passageiros: {passageiros}")
    lines.append("")
    lines.append("Emissões (kg CO₂):")
    lines.append(f"  Transporte (total): {transporte_total:.2f}")
    lines.append(f"  Transporte (por pessoa): {transporte_por_pessoa:.2f}")
    lines.append(f"  Hospedagem (total): {hotel_total:.2f}" if hotel_total > 0 else "  Hospedagem (total): 0.00")
    lines.append("-------------------------------")
    lines.append(f"  Total estimado: {total_kg:.2f} kg CO₂")
    lines.append(f"  Por pessoa: {total_por_pessoa:.2f} kg CO₂")
    lines.append("")
    lines.append("Dicas rápidas:")
    lines.append("- Prefira trem ou ônibus quando possível.")
    lines.append("- Compartilhe carro (mais passageiros reduz emissão por pessoa).")
    lines.append("- Prefira estadias simples ou diminua número de noites.")
    return "\n".join(lines)


def main():
    parser = argparse.ArgumentParser(description="Calculadora EcoTrip - estimativa de CO₂ para viagens")
    parser.add_argument("--distance", "--distancia", dest="distance", type=float, help="Distância em km")
    parser.add_argument("--mode", "--modo", dest="mode", type=str, help=f"Modo de transporte: {', '.join(EMISSION_FACTORS.keys())}")
    parser.add_argument("--passengers", "--passageiros", dest="passengers", type=int, default=1, help="Número de passageiros (inteiro >=1)")
    parser.add_argument("--roundtrip", "--ida-volta", dest="roundtrip", action="store_true", help="Marque se for ida e volta")
    parser.add_argument("--nights", "--noites", dest="nights", type=int, default=0, help="Número de noites de hospedagem")
    parser.add_argument("--hotel", dest="hotel", type=str, help=f"Tipo de hotel (opcional): {', '.join(HOTEL_FACTORS.keys())}")
    parser.add_argument("--json", dest="as_json", action="store_true", help="Imprimir resultado em JSON")
    parser.add_argument("--no-interactive", dest="no_interactive", action="store_true", help="Não entrar em modo interativo (falha se dados ausentes)")

    args = parser.parse_args()

    # Se nenhum argumento fornecido, entrar em modo interativo
    interactive = not any([args.distance, args.mode, args.nights, args.hotel]) and not args.no_interactive

    if interactive:
        # Modo interativo (compatível com seu script original)
        print("=== Calculadora EcoTrip (modo interativo) ===")
        try:
            dist_in = input("Distância (km) da viagem (apenas número): ").strip()
            distance = float(dist_in.replace(",", ".")) if dist_in else 0.0
            if distance < 0:
                raise ValueError()
        except Exception:
            print("Entrada inválida. Usando 0 km.")
            distance = 0.0

        print("Modos de transporte disponíveis:")
        print(", ".join(EMISSION_FACTORS.keys()))
        mode = input("Escolha o modo de transporte: ").strip() or "carro"

        try:
            pas_in = input("Número de passageiros (padrão 1): ").strip()
            passengers = int(pas_in) if pas_in else 1
            if passengers < 1:
                passengers = 1
        except Exception:
            passengers = 1

        ida_e_volta = parse_yesno(input("Ida e volta? (s/N): ") or "s")

        try:
            noites_in = input("Número de noites de estadia (0 para nenhum): ").strip()
            nights = int(noites_in) if noites_in else 0
            if nights < 0:
                nights = 0
        except Exception:
            nights = 0

        print("Tipos de hospedagem (opcional):", ", ".join(HOTEL_FACTORS.keys()))
        hotel = input("Tipo de hospedagem (enter para pular): ").strip() or None

    else:
        # modo não interativo: usar args (ou falhar)
        if args.distance is None or args.mode is None:
            parser.error("Em modo não interativo é obrigatório passar --distance e --mode (ou remover --no-interactive para usar interativo).")
        distance = max(0.0, args.distance)
        mode = args.mode
        passengers = max(1, args.passengers or 1)
        ida_e_volta = bool(args.roundtrip)
        nights = max(0, args.nights or 0)
        hotel = args.hotel

    # Cálculos (com tratamento de erros)
    try:
        transporte_total, transporte_por_pessoa = calcula_emissao_transporte(distance, mode, passengers, ida_e_volta)
    except ValueError as e:
        print(f"Erro: {e}")
        return 1

    try:
        hotel_total = calcula_emissao_hotel(nights, hotel)
    except ValueError as e:
        print(f"Erro: {e}")
        return 1

    total_kg = transporte_total + hotel_total
    total_por_pessoa = total_kg / max(1, passengers)

    if args.as_json:
        output = {
            "distance_km": round(distance, 3),
            "mode": mode,
            "round_trip": ida_e_volta,
            "passengers": passengers,
            "nights": nights,
            "hotel": hotel,
            "transport": {"total_kg": round(transporte_total, 4), "per_person_kg": round(transporte_por_pessoa, 4)},
            "hotel": {"total_kg": round(hotel_total, 4)},
            "total_kg": round(total_kg, 4),
            "per_person_kg": round(total_por_pessoa, 4)
        }
        print(json.dumps(output, ensure_ascii=False, indent=2))
    else:
        txt = formato_texto_saida(distance, mode, passengers, ida_e_volta, nights, hotel,
                                  transporte_total, transporte_por_pessoa, hotel_total, total_kg, total_por_pessoa)
        print(txt)
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
