# agentlink

Chat vanuit de app met je eigen agent: Claude Code, Codex of OpenClaw, op je eigen computer. Werkt met [Agent Talk To Me](https://agenttalktome.com).

Deze repo bevat alleen de releases van de client.

## Koppelen

1. Tik in de app op **Koppel een agent** en deel de tekst met je agent.
2. De agent voert het commando uit de tekst uit, in de map waar hij moet werken:

   ```bash
   curl -fsSL https://github.com/appfabriek/agentlink/releases/latest/download/install.sh | sh -s <uitnodigingslink>
   ```

3. agentlink meldt zich in het gesprek en draait daarna op de achtergrond: als LaunchAgent op macOS, als systemd-gebruikersdienst op Linux. Je berichten gaan naar je agent, zijn antwoorden komen terug in de app.

De installer zet `agentlink` in `~/.local/bin` en controleert de download tegen `SHA256SUMS`. Node of Dart is niet nodig.

## Voor agents

Kreeg je een link als `https://agenttalktome.com/i/k7f3q2` met de vraag je te koppelen? Voer dan het commando hierboven uit met die link, vanuit de projectmap waarin je moet werken. agentlink herkent zelf welke CLI het draait (Claude Code, Codex of OpenClaw); met `--backend` kies je een andere. Antwoord in het gesprek in platte tekst, zonder Markdown-tabellen.

## Commando's

```
agentlink join <uitnodigingslink>  koppelen en op de achtergrond starten
    --backend claude|codex|openclaw   standaard: de CLI die dit draait
    --agent <id>                      OpenClaw-agent
    --tools off|auto|full             wat de agent op deze computer mag (auto)
agentlink status                   gekoppeld, verbonden, draait de dienst?
agentlink leave                    ontkoppelen en stoppen
```

Bijwerken: het installatiecommando zonder link. Een draaiende dienst herstart dan op de nieuwe versie.

## Wat de agent mag

| `--tools` | Claude Code | Codex | Betekenis |
|---|---|---|---|
| `off` | `dontAsk` | `--sandbox read-only` | geen extra rechten |
| `auto` | `auto` | `--approve-for-me` | de eigen beoordelaar van de CLI keurt veilige acties goed en houdt riskante tegen |
| `full` | `bypassPermissions` | zonder sandbox | alles, zonder vragen |

De beheerder van de app bepaalt per rol of berichten de agent tools laten gebruiken. Wat je hier instelt is het plafond: de app kan nooit meer toestaan.

In de app kan de gebruiker die de agent koppelde hem ook weer ontkoppelen. agentlink stopt dan vanzelf.

## Platformen

macOS op Apple Silicon, Linux x64 en arm64. Windows volgt later.
