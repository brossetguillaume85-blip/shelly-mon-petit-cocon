import urllib.request
import urllib.error
from datetime import datetime, timedelta, timezone
import json
import re

# ============================================================
# CONFIGURATION
# ============================================================
ICAL_URL = "https://app.superhote.com/export-ics/2wAZuqS0Wf"
SHELLY_URL = "http://88.174.23.73:49200"  # IP publique + port redirigé
HOURS_BEFORE_CHECKIN = 4    # Allumer X heures avant arrivée
HOURS_AFTER_CHECKOUT = 1    # Éteindre X heures après départ
# ============================================================

def fetch_ical():
    """Récupère le calendrier iCal depuis Superhote"""
    req = urllib.request.Request(ICAL_URL)
    with urllib.request.urlopen(req, timeout=10) as response:
        return response.read().decode('utf-8')

def parse_ical_date(date_str):
    """Parse une date iCal (YYYYMMDD ou YYYYMMDDTHHmmssZ)"""
    date_str = date_str.strip()
    if 'T' in date_str:
        # Format datetime avec heure
        date_str = date_str.replace('Z', '')
        dt = datetime.strptime(date_str, '%Y%m%dT%H%M%S')
        return dt.replace(tzinfo=timezone.utc)
    else:
        # Format date seule — considère minuit UTC
        dt = datetime.strptime(date_str, '%Y%m%d')
        return dt.replace(tzinfo=timezone.utc)

def parse_reservations(ical_content):
    """Extrait les réservations depuis le contenu iCal"""
    reservations = []
    events = re.split(r'BEGIN:VEVENT', ical_content)
    
    for event in events[1:]:  # Skip le premier élément (header)
        dtstart_match = re.search(r'DTSTART(?:;VALUE=DATE)?(?:;TZID=[^:]+)?:(\S+)', event)
        dtend_match = re.search(r'DTEND(?:;VALUE=DATE)?(?:;TZID=[^:]+)?:(\S+)', event)
        summary_match = re.search(r'SUMMARY:(.+)', event)
        
        if dtstart_match and dtend_match:
            try:
                checkin = parse_ical_date(dtstart_match.group(1))
                checkout = parse_ical_date(dtend_match.group(1))
                summary = summary_match.group(1).strip() if summary_match else "Réservation"
                
                reservations.append({
                    'checkin': checkin,
                    'checkout': checkout,
                    'summary': summary
                })
            except Exception as e:
                print(f"Erreur parsing date: {e}")
                continue
    
    return reservations

def control_shelly(turn_on: bool):
    """Allume ou éteint le Shelly"""
    action = "true" if turn_on else "false"
    url = f"{SHELLY_URL}/rpc/Switch.Set?id=0&on={action}"
    state = "ALLUMAGE" if turn_on else "EXTINCTION"
    
    try:
        req = urllib.request.Request(url)
        with urllib.request.urlopen(req, timeout=10) as response:
            result = response.read().decode('utf-8')
            print(f"✅ {state} Shelly OK: {result}")
            return True
    except Exception as e:
        print(f"❌ Erreur {state} Shelly: {e}")
        return False

def get_shelly_status():
    """Récupère l'état actuel du Shelly"""
    url = f"{SHELLY_URL}/rpc/Switch.GetStatus?id=0"
    try:
        req = urllib.request.Request(url)
        with urllib.request.urlopen(req, timeout=10) as response:
            result = json.loads(response.read().decode('utf-8'))
            return result.get('output', None)
    except Exception as e:
        print(f"⚠️ Impossible de lire l'état Shelly: {e}")
        return None

def main():
    now = datetime.now(timezone.utc)
    print(f"\n{'='*50}")
    print(f"🕐 Vérification: {now.strftime('%d/%m/%Y %H:%M')} UTC")
    print(f"{'='*50}")
    
    # Récupération du calendrier
    try:
        ical_content = fetch_ical()
        print(f"✅ Calendrier Superhote récupéré")
    except Exception as e:
        print(f"❌ Erreur récupération calendrier: {e}")
        return
    
    # Parse des réservations
    reservations = parse_reservations(ical_content)
    print(f"📅 {len(reservations)} réservation(s) trouvée(s)")
    
    # Vérification état actuel du Shelly
    current_state = get_shelly_status()
    print(f"💡 État actuel Shelly: {'ON' if current_state else 'OFF' if current_state is not None else 'inconnu'}")
    
    # Logique de décision
    should_be_on = False
    reason = "Aucune réservation active"
    
    for res in reservations:
        checkin = res['checkin']
        checkout = res['checkout']
        
        # Fenêtre d'activation : 4h avant arrivée jusqu'à 1h après départ
        activation_start = checkin - timedelta(hours=HOURS_BEFORE_CHECKIN)
        activation_end = checkout + timedelta(hours=HOURS_AFTER_CHECKOUT)
        
        if activation_start <= now <= activation_end:
            should_be_on = True
            
            if now < checkin:
                mins_before = int((checkin - now).total_seconds() / 60)
                reason = f"Arrivée '{res['summary']}' dans {mins_before} min (préchauffage)"
            elif now < checkout:
                reason = f"Séjour en cours: '{res['summary']}'"
            else:
                mins_after = int((now - checkout).total_seconds() / 60)
                reason = f"Départ '{res['summary']}' il y a {mins_after} min (délai post-départ)"
            break
    
    print(f"🎯 Décision: {'ON' if should_be_on else 'OFF'} — {reason}")
    
    # Action si l'état doit changer
    if should_be_on and current_state == False:
        print("→ Allumage du Shelly...")
        control_shelly(True)
    elif not should_be_on and current_state == True:
        print("→ Extinction du Shelly...")
        control_shelly(False)
    elif current_state == should_be_on:
        print("→ Aucun changement nécessaire")
    else:
        # État inconnu — on applique la décision quand même
        print("→ État inconnu, application de la décision...")
        control_shelly(should_be_on)
    
    # Affichage des prochaines réservations
    print(f"\n📋 Prochaines réservations :")
    upcoming = [r for r in reservations if r['checkout'] > now]
    upcoming.sort(key=lambda x: x['checkin'])
    for r in upcoming[:3]:
        print(f"  • {r['summary']}: {r['checkin'].strftime('%d/%m %H:%M')} → {r['checkout'].strftime('%d/%m %H:%M')} UTC")

if __name__ == "__main__":
    main()
